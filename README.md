# awesome-math-libraries
Collection of math libraries in different programming languages

## Basic C++ guidelines

Existing cross platform SIMD implementations with style, of interest:
- [OpenMP](https://github.com/openmp) supported by mainstream compilers GCC/Clang/MSVC parallel SIMD
- [linalg](https://github.com/sgorsten/linalg/)
- [cmtd](github.com/uonrobotics/ctmd/)

The only portable usable n-dim serializable and arbitrarly alignable type of static extents seems to be a fixed std::array based type with added standard operators using `std::transform` and OpenMP. If available in C++26 compiler add `std::simd`, `std::submdspan` and static extent aligned type view later. Implementing Cross/Hodge/Laplace/Motor is necessary.

The following GCC/clang options are necessary to generate SIMD code output.
```sh
-march=native -mfpmath=<your simd> -ftree-vectorize -fopenmp -fopenmp-simd -O3 -std=gnu++26
```
Always use
```c
#ifdef _MSC_VER
#define INLINE __force_inline __flatten __declspec(nothrow) __declspec(noalias) inline
#else
#define INLINE __attribute__((always_inline,flatten,nothrow,const)) inline
#endif
```
for functions accessing the array/vector type.

Protoype example outline of n-dim fixed static extents arbitrary alignable serializable vector type:
```cpp
#include <cstdlib>
#include <cstdio>
#include <cstdint>
#include <cstddef>
#include <cmath>
#include <array>
#include <bit>
#include <utility>
#include <algorithm>
#include <functional>
#include <ranges>

#ifdef _MSC_VER
#define INLINE __force_inline __flatten __declspec(nothrow) __declspec(noalias) inline
#else
#define INLINE __attribute__((always_inline,flatten,nothrow,const)) inline
#endif

template<class T>
concept sca = std::integral<T> || std::floating_point<T>;

/* alignment mode */
enum class alg
{
        unk = 0 << 0,
        sca = 1 << 0, /* align to      1 * alignof(T) */
        vec = 1 << 1, /* align to N_POW2 * alignof(T) */
        mat = 1 << 2,
        ten = 1 << 3,
        qut = 1 << 4,
        mot = 1 << 5,
        sed = 1 << 6,
        oct = 1 << 7,
        std = 1 << 8, /* N == N_POW2 ? vec/mat/other : sca */
};

/* aligned fixed array buffer */
#pragma pack(push,1)
template<typename T, size_t N, enum alg A = alg::std, size_t N_POW2 = std::bit_ceil<size_t>(N)>
struct alignas((((N == N_POW2) || ((A != alg::sca) && ((A != alg::std) && (A != alg::unk)))) ? N_POW2 : 1) * alignof(T)) buf : std::array<T,N> {};
#pragma pack(pop)

/* entity type information */
enum class nfo
{
        pos = 0,
        dir,
        ori,
        col,
        fac,
        geo,
};

/* entity orientation */
enum class ori
{
        hor = 0,
        ver,
        dia,
        row,
        col,
};

template<typename T>
struct geo
{
};

/* arithmetic operator base */
template<typename T_THIS>
struct ari
{
        /* plus operators example */
        template<typename T_OTHER>
        INLINE T_THIS& operator+=(const T_OTHER &rhs) requires(!sca<T_OTHER>)
        {
                using this_type  = typeof((*(T_THIS*)this)[0]);
                using other_type = typeof(rhs[0]);
                static_assert(std::is_convertible_v<this_type,other_type>);

                #pragma omp simd
                for (std::size_t i = 0; i < std::min(rhs.size(), ((const T_THIS*)this)->size()); ++i)
                        (*(T_THIS*)this)[i] += this_type(rhs[i]);

                return (*(T_THIS*)this);
        }

        template<typename T_OTHER>
        INLINE T_THIS& operator+=(const T_OTHER &rhs) requires(sca<T_OTHER>)
        {
                using this_type  = typeof((*(T_THIS*)this)[0]);
                using other_type = typeof(rhs);
                static_assert(std::is_convertible_v<this_type,other_type>);

                #pragma omp simd
                for (std::size_t i = 0; i < ((const T_THIS*)this)->size(); ++i)
                        (*(T_THIS*)this)[i] += this_type(rhs);

                return (*(T_THIS*)this);
        }

        template<typename T_OTHER>
        friend INLINE T_THIS operator+(T_THIS lhs, const T_OTHER& rhs)
        {
                lhs += rhs;
                return lhs;
        }
};

/* aligned linear algebra vector type based on the types above */
template<typename T, size_t N, enum alg A = alg::std, nfo G = nfo::pos, ori O = ori::hor>
struct vec : buf<T,N,A>, ari<vec<T,N,A>>, geo<T>
{
        static constexpr enum alg alg = A;
        static constexpr enum nfo nfo = G;
        static constexpr enum ori ori = O;

        INLINE void draw() {}
        INLINE void print(ssize_t n = -1)
        {
              printf("[");
              n = n > -1 ? n : this->size();
              for(size_t i = 0; i < n; i++)
              {
                   printf("%+e%s", (*this)[i], (i != (n - 1)) ? " " : "");
              }
              printf("] ");
        }
        void print_alignment()
        {
              printf("%zu/%zu", alignof((*this)), sizeof((*this)));
        }
        vec(T src)
        {
                #pragma omp simd
                for(size_t i = 0; i < N; i++)
                        (*this)[i] = src;
        }
        vec(std::convertible_to<T> auto&&... src) requires(sizeof...(src) > 1) : std::array<T,N>{ std::forward<decltype(src)>(src)... }
        {
        }
};

int main(int argc, char** argv)
{
        vec<float,3>                   A = { 1, 2, 3 };
        vec<float,2,alg::vec,nfo::dir> B = { 1, 2 };
        vec<float,3>                   C = A + B;

        A.print();  A.print_alignment(); puts("");
        B.print(3); A.print_alignment(); puts("");
        C.print();  A.print_alignment(); puts("");
        exit(EXIT_SUCCESS);
}
```

Example of OpenMP dot product for C/C++
```c
#include <stdint.h>
#include <stddef.h>
#include <stdlib.h>
#include <stdio.h>
#include <sys/param.h>

#undef  countof
#define countof(a  ) (sizeof((a))/sizeof((a)[0]))
#define isarray(a  ) __builtin_choose_expr(__builtin_types_compatible_p(typeof((a)[0]) [], typeof((a))), true, false)

/* * * * * * * * * * * * * * *
 *    OpenMP dot product     *
 * * * * * * * * * * * * * * */
#define dot(a,b,n,i) \
({ \
typeof((a)[0]) dst = i; \
_Pragma("omp simd reduction(+:dst)") \
for(size_t j = 0; j < MIN(MIN(countof(a),countof(b)),n); j++) \
        dst += (a)[j] * (b)[j]; \
dst; \
})

#define dot2(a,b)         dot(a,b,2,0)
#define dot3(a,b)         dot(a,b,3,0)
#define dot4(a,b)         dot(a,b,4,0)

int main(int argc, char ** argv)
{
        float a[4] = {1,2,3,4};
        float b[4] = {5,6,7,8};

        printf("%f\n", dot4(a,b));
}
```
See: [Microsoft Visual Studio OpenMP SIMD](https://learn.microsoft.com/en-us/cpp/parallel/openmp/openmp-simd?view=msvc-180) for Microsoft specific compilers.

# Links
## Other C++ Math Libraries
- [HolyBlackCat/math](https://github.com/HolyBlackCat/math/)
- [xcmath](https://github.com/xcrtp/xcmath)
- [smath](https://github.com/slendidev/smath)
- [wykobi][45]
- [arrayfire][44]
- [libxsmm][43]
- [sleef][42]
- [petsc][40]
- [ctmd][32]
- [linalg][28]
- [lapack](https://github.com/Reference-LAPACK/lapack)
- [oneAPI/mkl](https://github.com/oneapi-src)
- [glm][35]
- [ITK][8]
- [vxl/vnl][9]
- [Terathon-Math-Library][10]
- [CML][20]
- [mr-math][11]
- [versor][12]
- [cutlass][33]
- [eve][34]
- [nicemath][36]
- [highway][46]
- [libsimdpp][47]

## C/C++ preprocessor essentials
- [HolyBlackCat/macro_sequence_for](https://github.com/HolyBlackCat/macro_sequence_for)

## C++ SIMD essentials
- [OpenMP][39]
- [Microsoft Visual Studio OpenMP SIMD](https://learn.microsoft.com/en-us/cpp/parallel/openmp/openmp-simd?view=msvc-180)
- [OpenGL/KHR][2]
- [Microsoft/DirectXMath][48]
- [SYCL Overview][23]
- [OpenCL][24]
- [CUDA C++ Programming Guide][25]
- [kokkos][37]
- [NumKong](https://github.com/ashvardanian/NumKong)

## Quake C++ Math Libraries
- [TrenchBroom/vm][5]
- [ericwa/ericw-tools][26]
- [PolyhedronStudio/Q2RTXPerimental][49]
- [a1batross/fakk2-sdk][21]
- [Prey2006/neo/idlib/math][22]
- [andrei-drexler/q321][27]
- [hogsy/chronon][29]
- [paulbaker/q3bsp][38]
- [Engine ports at NephatrineCode][41]

## Pascal Math Libraries
- [LMath](https://sourceforge.net/projects/lmath-library/)

[1]: https://isocpp.org/
[2]: https://github.com/KhronosGroup/OpenGL-Registry/blob/main/api/GL/glcorearb.h
[3]: https://github.com/jopadan/mlr/blob/main/include/mlr/array.hpp
[4]: https://github.com/jopadan/mlr/blob/main/include/mlr/vector.hpp

[5]: https://github.com/TrenchBroom/TrenchBroom/tree/master/lib/vm
[26]: https://github.com/ericwa/ericw-tools/blob/main/include/common/qvec.hh
[6]: https://github.com/quakeforge/quakeforge/tree/master/include/QF/simd
[7]: https://github.com/fte-team/fteqw
[27]: https://github.com/andrei-drexler/q321/blob/main/src/engine/math.h
[29]: https://github.com/hogsy/chronon/blob/master/qcommon/include/qcommon/math_vector.h

[8]: https://github.com/InsightSoftwareConsortium/ITK
[9]: https://github.com/vxl/vxl/tree/master/core/vnl
[10]: https://github.com/EricLengyel/Terathon-Math-Library
[11]: https://github.com/4J-company/mr-math/
[12]: https://github.com/wolftype/versor/

[13]: https://en.wikipedia.org/wiki/Permutation
[14]: https://maa.org/book/export/html/115646
[15]: https://en.wikipedia.org/wiki/Levi-Civita_symbol
[16]: https://en.wikipedia.org/wiki/Hodge_star_operator
[17]: https://catonif.github.io/cube/
[18]: https://github.com/prideout/par/
[19]: https://retrocomputing.stackexchange.com/questions/27400/what-is-the-most-accurate-way-to-map-6-bit-vga-palette-to-8-bit
[20]: https://github.com/demianmnave/CML
[21]: https://github.com/a1batross/fakk2-sdk/blob/master/source/source/qcommon/vector.h
[22]: https://github.com/FriskTheFallenHuman/Prey2006/blob/master/neo/idlib/math
[23]: https://www.khronos.org/sycl/
[24]: https://www.khronos.org/opencl/
[25]: https://docs.nvidia.com/cuda/cuda-c-programming-guide/
[28]: https://github.com/sgorsten/linalg/
[30]: https://jacquesheunis.com/post/rotors/
[31]: https://marctenbosch.com/quaternions/
[32]: https://github.com/uonrobotics/ctmd/
[33]: https://github.com/NVIDIA/cutlass/
[34]: https://github.com/jfalcou/eve/
[35]: https://github.com/g-truc/glm/
[36]: https://github.com/nicebyte/nicemath/
[37]: https://github.com/kokkos/kokkos/
[38]: https://www.paulsprojects.net/opengl/q3bsp/q3bsp.html
[39]: https://www.openmp.org/
[40]: https://petsc.org/
[41]: https://code.nephatrine.net/explore/repos/
[42]: https://sleef.org/
[43]: https://github.com/libxsmm/libxsmm/
[44]: https://github.com/arrayfire/arrayfire/
[45]: https://github.com/ArashPartow/wykobi/
[46]: https://github.com/google/highway/
[47]: https://github.com/p12tic/libsimdpp/
[48]: https://github.com/Microsoft/DirectXMath/
[49]: https://github.com/PolyhedronStudio/Q2RTXPerimental/tree/master/inc/shared/math/
