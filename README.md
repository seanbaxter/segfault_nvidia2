# NVIDIA driver segfault

Driver 455.46.01 
RTX 2060
Mint/Ubuntu 20.04

```
$ clang++ segfault.cxx -lgl3w -lglfw -DGOOD -o good
$ ./good
Loading shader _Z9frag_mainI15tracer_engine_tISt4pairI15sphere_tracer_t16segment_tracer_tE7blobs_tEEvv from good.spv
Specialized shader

$ clang++ segfault.cxx -lgl3w -lglfw -DBAD -o bad
$ ./bad
Loading shader _Z9frag_mainI15tracer_engine_tISt4pairI15sphere_tracer_t16segment_tracer_tE7blobs_tEEvv from bad.spv
Segmentation fault (core dumped)
```

## Context

In response to [3149892](https://developer.nvidia.com/nvidia_bug/3149892) I refactored my SPIR-V codegen to only use `Offset`- and `ArrayStride`-decorated types when instantiated on layout storage classes like `Uniform` and `StorageBuffer`. On trivial object copies, a recursive subobject-wise assignments copies data between these two SPIR-V types that correspond to the same C++ type.

Before this refactor, with the compiler that allegedly violated 3149892, this shader worked fine. That's `good.spv`. 

This is what it draws:

![segment tracer](https://raw.githubusercontent.com/seanbaxter/shaders/master/images/segment_tracer.png)

I can confirm that the segfault is caused by the driver failing to copy the decorated structure from a `Uniform` variable to its corresponding logical structure in `Function` storage.

```cpp
template<typename shader_t>
[[spirv::frag(lower_left)]]
void frag_main() {
  fragColor = shader_ubo<shader_t>.render(glfrag_FragCoord.xy, uniforms);
}
```

The usual fragment shader entry point `frag_main` casts its UBO to `shader_t`, which in this shader is include an std::pair holding data for both sphere tracing and segment tracing. This implementation still works. Since SPIR-V doesn't allow `Uniform` storage class `this` pointers, the compiler inlines the `render` code and everything using the UBO.

```cpp
template<typename shader_t>
[[spirv::frag(lower_left)]]
void frag_main() {
  shader_t object = shader_ubo<shader_t>;
  fragColor = object.render(glfrag_FragCoord.xy, uniforms);
}
```

In the modified code that breaks with the 3149892 workaround, the UBO is copied to a `Function` storage class object, and the render code is invoked as a member function on that. This code is fine pre-workaround.

```
        %468 = OpAccessChain %_ptr_Uniform_tracer_engine_t_std__pair_sphere_tracer_t__segment_tracer_t___blobs_t_ %_Z10shader_uboI15tracer_engine_tISt4pairI15sphere_tracer_t16segment_tracer_tE7blobs_tEE %int_0
        %469 = OpLoad %tracer_engine_t_std__pair_sphere_tracer_t__segment_tracer_t___blobs_t_ %468
        %470 = OpCompositeExtract %int %469 0
        %471 = OpCompositeExtract %std__pair_sphere_tracer_t__segment_tracer_t_ %469 1
        %472 = OpCompositeExtract %sphere_tracer_t %471 1
        %473 = OpCompositeExtract %float %472 0
        %474 = OpCompositeConstruct %sphere_tracer_t_0 %473
        %475 = OpCompositeExtract %segment_tracer_t %471 2
        %476 = OpCompositeExtract %float %475 0
        %477 = OpCompositeExtract %float %475 1
        %478 = OpCompositeConstruct %segment_tracer_t_0 %476 %477
        %480 = OpCompositeConstruct %std__pair_sphere_tracer_t__segment_tracer_t__0 %479 %474 %478
        %481 = OpCompositeExtract %blobs_t %469 2
        %482 = OpCompositeExtract %float %481 0
        %483 = OpCompositeExtract %float %481 1
        %484 = OpCompositeConstruct %blobs_t_0 %482 %483
        %485 = OpCompositeExtract %v3float %469 3
        %486 = OpCompositeExtract %v3float %469 4
        %487 = OpCompositeExtract %v3float %469 5
        %488 = OpCompositeExtract %v3float %469 6
        %489 = OpCompositeExtract %v3float %469 7
        %490 = OpCompositeConstruct %tracer_engine_t_std__pair_sphere_tracer_t__segment_tracer_t___blobs_t__0 %470 %480 %484 %485 %486 %487 %488 %489
               OpStore %463 %490
```

I suspect the problem is how the driver handles the empty base class of std::pair. See:

```
               OpMemberDecorate %std__pair_sphere_tracer_t__segment_tracer_t_ 0 Offset 0
               OpMemberDecorate %std__pair_sphere_tracer_t__segment_tracer_t_ 1 Offset 0
               OpMemberDecorate %std__pair_sphere_tracer_t__segment_tracer_t_ 2 Offset 4

%std____pair_base_sphere_tracer_t__segment_tracer_t_ = OpTypeStruct
%sphere_tracer_t = OpTypeStruct %float
%segment_tracer_t = OpTypeStruct %float %float
%std__pair_sphere_tracer_t__segment_tracer_t_ = OpTypeStruct %std____pair_base_sphere_tracer_t__segment_tracer_t_ %sphere_tracer_t %segment_tracer_t
```

The first member of `std__pair_sphere_tracer_t__segment_tracer_t_` is an empty struct. See the std::pair implementation:

```

  template<typename _U1, typename _U2> class __pair_base
  {
#if __cplusplus >= 201103L
    template<typename _T1, typename _T2> friend struct pair;
    __pair_base() = default;
    ~__pair_base() = default;
    __pair_base(const __pair_base&) = default;
    __pair_base& operator=(const __pair_base&) = delete;
#endif // C++11
  };

  template<typename _T1, typename _T2>
    struct pair
    : private __pair_base<_T1, _T2>
    {
      typedef _T1 first_type;    ///< The type of the `first` member
      typedef _T2 second_type;   ///< The type of the `second` member

      _T1 first;                 ///< The first member
      _T2 second;                ///< The second member

      ...
    };
```

It's more logical to keep the empty base class in the struct definition, since code may need to form a pointer to the `__pair_base`, and SPIR-V's lack of pointers means it can't really bitcast the pointer to the variable to the type of the base class. Empty structs are explicitly supported by SPIR-V: "Declare a new structure type: an aggregate of zero or more potentially heterogeneous members."

Again, I don't know if this is the root cause of the segfault. It's just a hunch.