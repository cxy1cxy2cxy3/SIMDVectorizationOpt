# Disclaimer
此项目仅供学习与交流使用，请遵循MIT协议，如果您在任何项目中使用相关代码，请保留此项目的LICENSE文件。

# SIMD Vectorization Demo

This demo is made by [Wang Lipeng](https://github.com/WANG-lp) and [CHE Yulin](https://github.com/CheYulin) from Hong Kong University of Science and Technology. Feel free to give this repo a star if you like it.

intel intrinsics guide: see https://software.intel.com/sites/landingpage/IntrinsicsGuide/

bechmarking CPU info: see [src/playground](src/playground), using single-thread

## Tokenizing

/ | type
--- | ---
input  | form body
output | dubbo package

algorithms | elapsed time
--- | ---
serial transform time | 380 ms
sse4 transform time | 240 ms
avx2 transform time | 90 ms

* workaround for `__tzcnt_u32` (requiring bmi)  in sse4 (handle 16bits currently, else modify zero if-sentence)

```cpp
inline int __tzcnt_u32_using_popcnt_cmpeq(unsigned int x) {
    int ret_cnt = 0;
    int half_bits = x & (0xff);
    if (half_bits == 0) {
        half_bits = (x >> 8) & (0xff);
        ret_cnt = 8;
    }

    __m128i pivot_u = _mm_set1_epi16(half_bits);
    __m128i inspected_ele = _mm_set_epi16(
            0xff, 0x7f, 0x3f, 0x1f, 0xf, 0x7, 0x3, 0x1);
    __m128i trunc_pivot_u = _mm_and_si128(pivot_u, inspected_ele);

    __m128i pivot_new = _mm_set1_epi16(0);
    __m128i cmp_res = _mm_cmpeq_epi16(trunc_pivot_u, pivot_new);

    int mask = _mm_movemask_epi8(cmp_res); // 16 bits
    ret_cnt += _mm_popcnt_u32(mask) >> 1;
    return ret_cnt;
}
```

* idea: advance with simd cmp and tzcnt

```cpp
        // 1st: advance fo find first '='
        while (true) {
            __m128i pivot_u = _mm_set1_epi8('=');
            __m128i inspected_ele = _mm_loadu_si128((__m128i *) (str + i));
            __m128i cmp_res = _mm_cmpeq_epi8(pivot_u, inspected_ele);
            int mask = _mm_movemask_epi8(cmp_res); // 16 bits

            int advance = (mask == 0 ?
                           16 : __tzcnt_u32_using_popcnt_cmpeq(mask));
            i += advance;
            if (advance < 16) { break; }
        }
        off[next] = i;
        next++;
```

* details: see [src/fast_tokenizer/tokenizer.cpp](src/fast_tokenizer/tokenizer.cpp), [src/playground/test_split.cpp](src/playground/test_split.cpp)

## Base64 Decoding/Encoding (i.e, Serialization/Deserialization)

* base64 encoding includes 1) unpacking and 2) ascii translation (to)

* base64 decoding includes 1) ascii translation (from) and 2) packing

* the translation table

base64 | ascii-dec | ascii-hex | offset (diff)
---    | ---   | --- | ---
0-25   | 65-90  | 0x41-0x5a | 65
26-51  | 97-122 | 0x61-0x7a | 71
52-61  | 48-57  | 0x30-0x39 | -4
62     | 43, i.e., '+' | 0x2b      | -19
63     | 47, i.e., '/' | 0x2f      | -16
others | invalid            | invalid | invalid

### Base64 Decoding Vectorization

*ascii code to base64 translation vectorization (simplified)*

```cpp
        // attention: each SIMD lane is 128-bits
        const __m256i lut_roll = _mm256_setr_epi8(
            0,   16,  19,   4, -65, -65, -71, -71,
            0,   0,   0,   0,   0,   0,   0,   0,
            0,   16,  19,   4, -65, -65, -71, -71,
            0,   0,   0,   0,   0,   0,   0,   0
        );
        const __m256i mask_2F = _mm256_set1_epi8(0x2f);

        // 1st: shift right for 4 32bit-wise-elements in each SIMD lane
        // shift to get high 4bits
        __m256i hi_nibbles  = _mm256_srli_epi32(str, 4);
        
        // 2nd: cmpeq to distinguish 0x2f ('/') from 0x2b ('+') 
        // when the byte is 0x2f, the result byte in eq_2F is 0xff (-1)  
        const __m256i eq_2F = _mm256_cmpeq_epi8(str, mask_2F);

        // 3rd: add -1 when the byte is 0x2f, 
        // lookup lut_roll for offsets to add, shuffling in each SIMD Lane
        // shuffling using the previous high 4-bits in the byte (see 1st step)
        const __m256i roll  = _mm256_shuffle_epi8(lut_roll, 
                                _mm256_add_epi8(eq_2F, hi_nibbles));
        
        // 4th: add offsets
        str = _mm256_add_epi8(str, roll);
```

### Benchmark

* implementations

benchmark main file: [src/playground/test_serialization.cpp](src/playground/test_serialization.cpp)

algorithms | link
--- | ---
naive-comp   | [src/naive_base64/naive_base64.cpp](src/naive_base64/naive_base64.cpp)
naive-lookup | [src/naive_base64/naive_base64.cpp](src/naive_base64/naive_base64.cpp)
chromium     | [src/fast_base64/chromiumbase64.c](src/fast_base64/chromiumbase64.c)
fast-avx2    | [src/fast_base64/fastavxbase64.c](src/fast_base64/fastavxbase64.c)

* measure: we report TPS (throughput including both serialization and deserialization time)

algorithms | idea
--- | ---
naive-comp   | ascii-base64 translation using computations
naive-lookup | ascii-base64 translation using two look-up tables
chromium     | use separate lookup tables for saving some computations
fast-avx2    | apply avx2, and serial fallback using chromium


* message len 200

algorithms | TPS-total | TPS-serializtion | TPS-deserializtion
--- | ---  | --- | --- 
naive_comp | 201.448 MB/s | 467.150 MB/s | 354.179 MB/s
naive_lookup | 490.444 MB/s | 1080.221 MB/s | 898.286 MB/s
chromium | 493.397 MB/s | 1039.237 MB/s | 939.390 MB/s
avx2_chromium | 887.260 MB/s | 1838.769 MB/s | 1714.608 MB/s

* message len 80

algorithms | TPS-total | TPS-serializtion | TPS-deserializtion
--- | ---  | --- | --- 
naive_comp | 149.235 MB/s | 288.609 MB/s | 309.029 MB/s
naive_lookup | 394.224 MB/s | 740.423 MB/s | 843.135 MB/s
chromium | 409.404 MB/s | 725.954 MB/s | 938.900 MB/s
avx2_chromium | 556.984 MB/s | 974.215 MB/s | 1300.531 MB/s

* message len 50

algorithms | TPS-total | TPS-serializtion | TPS-deserializtion
--- | ---  | --- | --- 
naive_comp | 115.787 MB/s | 218.357 MB/s | 246.493 MB/s
naive_lookup | 330.488 MB/s | 598.695 MB/s | 737.719 MB/s
chromium | 335.035 MB/s | 595.675 MB/s | 765.698 MB/s
avx2_chromium | 385.963 MB/s | 656.360 MB/s | 936.887 MB/s


