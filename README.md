# awesome-math-libraries
Collection of math libraries in different programming languages

## Essential Readings
### Types
#### Basics
1. use `INLINE`for and `OpenMP` in functions accessing the type.
2. use `-ftree-vectorize -fopenmp -fopenmp-simd -O3 -march=<your target ARCH> -mfpmath=<your SIMD> -std=[gnu|c++]23`

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
#define countof(x) sizeof(x)/sizeof((x)[0]) 
#define isarray(a) __builtin_choose_expr(__builtin_types_compatible_p(typeof((a)[0]) [], typeof((a))), true, false)

/* WARNING: vec_ext_countof(a) return different results in GCC/LLVM for non-power two element count */
/* TODO: find a way to return vector element count for non-power of two vectors in GCC */
/* vec_ext(t,n) for LLVM SIMD Vector Extension */
#if defined(__clang__)
#pragma pack(push,1)
#define vec_ext(T,N) typeof(T __attribute__((ext_vector_type(N))))
#pragma pack(pop)

#define rc_vec(src,n) (*(vec(typeof((src)[0]),n)*)__builtin_addressof(src))
#define rc_vec_ext(src,n) (*(vec_ext(typeof((src)[0]),n)*)__builtin_addressof(src))
#define vec_ext_countof(a) __builtin_vectorelements(rc_vec_ext(a,countof(a)))
#define isvector(a) !isarray(a) && \
__builtin_choose_expr( \
bitceil(vec_ext_countof(a)) == countof(a),true,false)

/* vec_ext(t,n) for GCC SIMD Vector Extension */
#elif defined(__GNUC__)
#pragma pack(push,1)
#define vec_ext(T,N) typeof(T __attribute__((vector_size(bitceil(N) * alignof(T)))))
#pragma pack(pop)

#define rc_vec(src,n) (*(vec(typeof((src)[0]),n)*)&src)
#define rc_vec_ext(src,n) (*(vec_ext(typeof((src)[0]),n)*)&src)
#define vec_ext_countof(a) countof(a)
#define isvector(a) !isarray(a) && \
_Generic(__builtin_convertvector(rc_vec_ext(a,vec_ext_countof(a)), \
         vec_ext(float,bitceil(vec_ext_countof(a)))), \
         vec_ext(float,bitceil(vec_ext_countof(a))): true, \
         default: false)

/* vec_ext(T,N) for Microsoft Visual Studio C++ compiler */
#elif defined(_MSC_VER)
#define vec_ext(T,N) typeof(T __declspec((align(sizeof(T)*bitceil(N))))[bitceil(N)])
#warn "Your compiler doens't support vector extensions."
#warn "Using aligned arrays without operators instead."

/* vec_ext(T,N) for other compilers */
#else
#define vec_ext(T,N) typeof(T __attribute__((aligned(sizeof(T)*bitceil(N))))[bitceil(N)])
#warn "Your compiler doens't support vector extensions."
#warn "Using aligned arrays without operators instead."
#endif
```
Dot product example:
```c
#include "array.h"

#define dot(_a,_b,_t,_n,_i) \
({ \
        _t _dst = (_t)_i; \
        static_assert((isarray(_a) || isvector(_a)),"ERROR: dot - a is not of array or vector type!"); \
        static_assert((isarray(_b) || isvector(_b)),"ERROR: dot - b is not of array or vector type!"); \
        size_t len = (size_t)llabs((ssize_t)_n); \
        len = MIN(MIN(!isarray(_a) ? vec_ext_countof(_a) : countof(_a), \
                      !isarray(_b) ? vec_ext_countof(_b) : countof(_b)), \
                  (ssize_t)_n>0?len:0); \
        _Pragma("omp simd reduction(+:_dst)") \
        for(size_t j = 0; j < len; j++) \
               _dst += (_t)((_a)[j]) * (_t)((_b)[j]); \
	_dst; \
})

typedef float real;
#define array(t,n) alignas((n == bitceil(n) ? n : 1) * alignof(t)) typeof(t[n])

#define dot3(a,b) dot((a),(b),real,3,0)
#define dot4(a,b) dot((a),(b),real,4,0)

int main(int argc, char** argv)
{
        vec_ext(real,3) a = { 1,  3, -5};
        array(real,3)   b = { 4, -2, -1};
        printf("%+e %+e\n", dot3(a,b),dot3(b,a));
        exit(EXIT_SUCCESS);
}
```
#### C++ `std::array<T,N>`
```cpp
template<typename T,size_t N, size_t N_POW2 = std::bit_ceil<size_t>(N)>
struct alignas((N == N_POW2 ? N : 1) * alignof(T)) vec<T,N> : std::array<T,N> {};
static_assert(countof(vec<T,N>) == N);
static_assert(sizeof(vec<T,N>) == (N * sizeof(T)));
```
#### GCC `typeof(T __attribute__((vector_size(N_POW2 * sizeof(T)))))`
```cpp
#define vec(T,N) typeof(T __attribute__((vector_size(bitceil(N)))))
static_assert(countof(vec(T,N)) == bitceil(N));
static_assert(sizeof(vec(T,N)) == (bitceil(N) * sizeof(T)));
```
#### LLVM `typeof(T __attribute__((ext_vector_type(N))))`
```cpp
#define vec(T,N) typeof(T __attribute__((ext_vector_type(N))))
static_assert(__builtin_vectorelements(vec(T,N)) == N && countof(vec(T,N)) == bitceil(N));
static_assert(sizeof(vec(T,N)) == (bitceil(N) * sizeof(T)));
```
#### C++ std::simd
#### C++ std::submdspan
#### C array
#### C struct
#### C/C++ SIMD platform specific intrinsic builtins

### API
- [OpenMP](https://www.openmp.org/)
- [OpenCL](https://www.khronos.org/opencl/)
- [OpenGL/KHR](https://github.com/KhronosGroup/OpenGL-Registry/blob/main/api/GL/glcorearb.h)
- [SYCL Overview](https://www.khronos.org/sycl/)
- [Microsoft/DirectXMath](https://github.com/Microsoft/DirectXMath/)
- [Microsoft Visual Studio OpenMP SIMD](https://learn.microsoft.com/en-us/cpp/parallel/openmp/openmp-simd?view=msvc-180)
- [Vectorization optimization in GCC](https://developers.redhat.com/articles/2023/12/08/vectorization-optimization-gcc#other_examples)
### C++
- [ISO C++](https://isocpp.org/)
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
- [libigl](https://github.com/libigl/libigl)
- [cgal](https://github.com/CGAL/cgal)
- [qhull](https://github.com/qhull/qhull)
- [muparser](https://github.com/beltoforion/muparser)
### NVIDIA
- [cutlass](https://github.com/NVIDIA/cutlass/)
- [CUDA C++ Programming Guide](https://docs.nvidia.com/cuda/cuda-c-programming-guide/)
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

