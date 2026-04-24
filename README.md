# awesome-math-libraries
Collection of math libraries in different programming languages

## Essential Readings
### Types
#### Basics
```cpp
enum class alg
{
        unk = 0 << 0,
        sca = 1 << 0, /* align to            1 * alignof(T)            */
        vec = 1 << 1, /* align to       N_POW2 * alignof(T)            */
        mat = 1 << 2, /* align to bitceil(COLS * alignof(vec<T,ROWS>)) */
        ten = 1 << 3,
        qut = 1 << 4,
        mot = 1 << 5,
        sed = 1 << 6,
        oct = 1 << 7,
        std = 1 << 8, /* N == N_POW2 ? vec/mat/other : sca             */
};

#ifdef _MSC_VER
#define INLINE __force_inline __flatten __declspec(noalias) __declspec(nothrow) inline
#else
#define INLINE __attribute__((always_inline),(nothrow),(const),(flatten)) inline
#endif

#define BITOP_RUP01__(x) (             (x) | (             (x) >>  1))
#define BITOP_RUP02__(x) (BITOP_RUP01__(x) | (BITOP_RUP01__(x) >>  2))
#define BITOP_RUP04__(x) (BITOP_RUP02__(x) | (BITOP_RUP02__(x) >>  4))
#define BITOP_RUP08__(x) (BITOP_RUP04__(x) | (BITOP_RUP04__(x) >>  8))
#define BITOP_RUP16__(x) (BITOP_RUP08__(x) | (BITOP_RUP08__(x) >> 16))

#define bitceil(x) (const uint32_t)(BITOP_RUP16__(((uint32_t)(x)) - 1) + 1)
```
#### C++ `std::array<T,N>` (aligned if N==N_POW2) using OpenMP and INLINE for members/operators
```cpp
template<typename T,size_t N, size_t N_POW2 = std::bit_ceil<size_t>(N)>
struct alignas((N == N_POW2 ? N : 1) * alignof(T)) vec<T,N> : std::array<T,N> {};
static_assert(countof(vec<T,N>) == N);
static_assert(sizeof(vec<T,N>) == (N * sizeof(T)));
```
#### GCC `typeof(T __attribute__((vector_size(N_POW2 * sizeof(T)))))` with OpenMP, preprocessor and INLINE
```cpp
#define vec(T,N) typeof(T __attribute__((vector_size(bitceil(N)))))
static_assert(countof(vec(T,N)) == bitceil(N));
static_assert(sizeof(vec(T,N)) == (bitceil(N) * sizeof(T)));
```
#### LLVM `typeof(T __attribute__((ext_vector_type(N))))` with OpenMP, preprocessor and INLINE
```cpp
#define vec(T,N) typeof(T __attribute__((ext_vector_type(N))))
static_assert(__builtin_elementcount(vec(T,N)) == N && countof(vec(T,N)) == bitceil(N));
static_assert(sizeof(vec(T,N)) == (bitceil(N) * sizeof(T)));
```
##### C array (aligned if N==N_POW2) with OpenMP, preprocessor and INLINE
##### C struct (aligned if N==N_POW2) with OpenMP, preprocessor and INLINE
##### C++ std::simd
##### C++ std::submdspan
##### C/C++ SIMD platform specific intrinsic builtins

### API
- [OpenMP](https://www.openmp.org/)
- [OpenCL](https://www.khronos.org/opencl/)
- [OpenGL/KHR](https://github.com/KhronosGroup/OpenGL-Registry/blob/main/api/GL/glcorearb.h)
- [SYCL Overview](https://www.khronos.org/sycl/)
- [Microsoft/DirectXMath](https://github.com/Microsoft/DirectXMath/)
- [Microsoft Visual Studio OpenMP SIMD](https://learn.microsoft.com/en-us/cpp/parallel/openmp/openmp-simd?view=msvc-180)
### C++
- [ISO C++](https://isocpp.org/)
- [CUDA C++ Programming Guide](https://docs.nvidia.com/cuda/cuda-c-programming-guide/)
- [kokkos](https://github.com/kokkos/kokkos/)
### C/C++ Preprocessor
- [HolyBlackCat/math](https://github.com/HolyBlackCat/math/)
- [HolyBlackCat/macro_sequence_for](https://github.com/HolyBlackCat/macro_sequence_for)

## C++ Math Libraries
### General
- [xcmath](https://github.com/xcrtp/xcmath)
- [smath](https://github.com/slendidev/smath)
- [wykobi](https://github.com/ArashPartow/wykobi/)
- [arrayfire](https://github.com/arrayfire/arrayfire/)
- [libxsmm](https://github.com/libxsmm/libxsmm/)
- [sleef](https://sleef.org/)
- [petsc](https://petsc.org/)
- [ctmd](https://github.com/uonrobotics/ctmd/)
- [linalg](https://github.com/sgorsten/linalg/)
- [lapack](https://github.com/Reference-LAPACK/lapack)
- [NumKong](https://github.com/ashvardanian/NumKong)
- [version2](https://github.com/vectorclass/version2)
- [oneAPI/mkl](https://github.com/oneapi-src)
- [glm](https://github.com/g-truc/glm/)
- [ITK](https://github.com/InsightSoftwareConsortium/ITK)
- [vxl/vnl](https://github.com/vxl/vxl/tree/master/core/vnl)
- [Terathon-Math-Library](https://github.com/EricLengyel/Terathon-Math-Library)
- [CML](https://github.com/demianmnave/CML)
- [mr-math](https://github.com/4J-company/mr-math/)
- [versor](https://github.com/wolftype/versor/)
- [cutlass](https://github.com/NVIDIA/cutlass/)
- [eve](https://github.com/jfalcou/eve/)
- [nicemath](https://github.com/nicebyte/nicemath/)
- [highway](https://github.com/google/highway)
- [libsimdpp](https://github.com/p12tic/libsimdpp/)
- [kfr](https://github.com/kfrlib/kfr)
- [prideout/par](https://github.com/prideout/par/)
- [eigen](https://gitlab.com/libeigen/eigen)
- [armadillo](https://github.com/conradsnicta/armadillo-code)
- [ensmallen](https://github.com/mlpack/ensmallen)
- [bandicoot](https://gitlab.com/bandicoot-lib/bandicoot-code)
- [flint](https://github.com/flintlib/flint)
### Quake
- [TrenchBroom/vm](https://github.com/TrenchBroom/TrenchBroom/tree/master/lib/vm)
- [ericwa/ericw-tools](https://github.com/ericwa/ericw-tools/blob/main/include/common/qvec.hh)
- [PolyhedronStudio/Q2RTXPerimental](https://github.com/PolyhedronStudio/Q2RTXPerimental/tree/master/inc/shared/math/)
- [a1batross/fakk2-sdk](https://github.com/a1batross/fakk2-sdk/blob/master/source/source/qcommon/vector.h)
- [Prey2006/neo/idlib/math](https://github.com/FriskTheFallenHuman/Prey2006/blob/master/neo/idlib/math)
- [andrei-drexler/q321](https://github.com/andrei-drexler/q321/blob/main/src/engine/math.h)
- [hogsy/chronon](https://github.com/hogsy/chronon/blob/master/qcommon/include/qcommon/math_vector.h)
- [paulbaker/q3bsp](https://www.paulsprojects.net/opengl/q3bsp/q3bsp.html)
- [fte-team/fteqw](https://github.com/fte-team/fteqw)
- [quakeforge/quakeforge](https://github.com/quakeforge/quakeforge/tree/master/include/QF/simd)
- [Engine ports at NephatrineCode](https://code.nephatrine.net/explore/repos/)

## Pascal Math Libraries
- [LMath](https://sourceforge.net/projects/lmath-library/)

