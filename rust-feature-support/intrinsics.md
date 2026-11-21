{% raw %}
# イントリンシック {#intrinsics}

下の表は、Rust のイントリンシックに対する Kani の現在のサポート状況をまとめたものです。
サポートの水準は、[Rust のフィーチャのサポート](../rust-feature-support.html)での示し方と同じように定義します。
 * **対応**: そのイントリンシックは完全にサポートされています。問題は把握していません。
 * **部分対応**: そのイントリンシックは少なくとも部分的にはサポートされています。いくつかの問題を
 把握しています。
 * **非対応**: そのイントリンシックはサポートされていません。

一般に、サポートされていないイントリンシックのコード生成は、
[Rust のフィーチャのサポート - サポートされていない
フィーチャのコード生成](../rust-feature-support.html#code-generation-for-unsupported-features)で説明した規則に従います。

下の表に載っていないイントリンシックは、サポートされていないとみなします。
コードがサポートされていないイントリンシックに依存している場合は、
[フィーチャのリクエストを作成](https://github.com/model-checking/kani/issues/new?assignees=&labels=%5BC%5D+Feature+%2F+Enhancement&template=feature_request.md&title=)してください。

### コンパイラのイントリンシック {#compiler-intrinsics}

名前 | サポート | 備考 |
--- | --- | --- |
abort | 対応 | |
abort_immediate | 対応 | |
add_with_overflow | 対応 | |
arith_offset | 対応 | |
assert_inhabited | 対応 | |
assert_uninit_valid | 対応 | |
assert_zero_valid | 対応 | |
assume | 対応 | |
atomic_and_seqcst | 部分対応 | [アトミック](#atomics)を参照 |
atomic_and_acquire | 部分対応 | [アトミック](#atomics)を参照 |
atomic_and_acqrel | 部分対応 | [アトミック](#atomics)を参照 |
atomic_and_release | 部分対応 | [アトミック](#atomics)を参照 |
atomic_and_relaxed | 部分対応 | [アトミック](#atomics)を参照 |
atomic_cxchg_acqrel_acquire | 部分対応 | [アトミック](#atomics)を参照 |
atomic_cxchg_acqrel_relaxed | 部分対応 | [アトミック](#atomics)を参照 |
atomic_cxchg_acqrel_seqcst | 部分対応 | [アトミック](#atomics)を参照 |
atomic_cxchg_acquire_acquire | 部分対応 | [アトミック](#atomics)を参照 |
atomic_cxchg_acquire_relaxed | 部分対応 | [アトミック](#atomics)を参照 |
atomic_cxchg_acquire_seqcst | 部分対応 | [アトミック](#atomics)を参照 |
atomic_cxchg_relaxed_acquire | 部分対応 | [アトミック](#atomics)を参照 |
atomic_cxchg_relaxed_relaxed | 部分対応 | [アトミック](#atomics)を参照 |
atomic_cxchg_relaxed_seqcst | 部分対応 | [アトミック](#atomics)を参照 |
atomic_cxchg_release_acquire | 部分対応 | [アトミック](#atomics)を参照 |
atomic_cxchg_release_relaxed | 部分対応 | [アトミック](#atomics)を参照 |
atomic_cxchg_release_seqcst | 部分対応 | [アトミック](#atomics)を参照 |
atomic_cxchg_seqcst_acquire | 部分対応 | [アトミック](#atomics)を参照 |
atomic_cxchg_seqcst_relaxed | 部分対応 | [アトミック](#atomics)を参照 |
atomic_cxchg_seqcst_seqcst | 部分対応 | [アトミック](#atomics)を参照 |
atomic_cxchgweak_acqrel_acquire | 部分対応 | [アトミック](#atomics)を参照 |
atomic_cxchgweak_acqrel_relaxed | 部分対応 | [アトミック](#atomics)を参照 |
atomic_cxchgweak_acqrel_seqcst | 部分対応 | [アトミック](#atomics)を参照 |
atomic_cxchgweak_acquire_acquire | 部分対応 | [アトミック](#atomics)を参照 |
atomic_cxchgweak_acquire_relaxed | 部分対応 | [アトミック](#atomics)を参照 |
atomic_cxchgweak_acquire_seqcst | 部分対応 | [アトミック](#atomics)を参照 |
atomic_cxchgweak_relaxed_acquire | 部分対応 | [アトミック](#atomics)を参照 |
atomic_cxchgweak_relaxed_relaxed | 部分対応 | [アトミック](#atomics)を参照 |
atomic_cxchgweak_relaxed_seqcst | 部分対応 | [アトミック](#atomics)を参照 |
atomic_cxchgweak_release_acquire | 部分対応 | [アトミック](#atomics)を参照 |
atomic_cxchgweak_release_relaxed | 部分対応 | [アトミック](#atomics)を参照 |
atomic_cxchgweak_release_seqcst | 部分対応 | [アトミック](#atomics)を参照 |
atomic_cxchgweak_seqcst_acquire | 部分対応 | [アトミック](#atomics)を参照 |
atomic_cxchgweak_seqcst_relaxed | 部分対応 | [アトミック](#atomics)を参照 |
atomic_cxchgweak_seqcst_seqcst | 部分対応 | [アトミック](#atomics)を参照 |
atomic_fence_seqcst | 部分対応 | [アトミック](#atomics)を参照 |
atomic_fence_acquire | 部分対応 | [アトミック](#atomics)を参照 |
atomic_fence_acqrel | 部分対応 | [アトミック](#atomics)を参照 |
atomic_fence_release | 部分対応 | [アトミック](#atomics)を参照 |
atomic_load_seqcst | 部分対応 | [アトミック](#atomics)を参照 |
atomic_load_acquire | 部分対応 | [アトミック](#atomics)を参照 |
atomic_load_relaxed | 部分対応 | [アトミック](#atomics)を参照 |
atomic_load_unordered | 部分対応 | [アトミック](#atomics)を参照 |
atomic_max_seqcst | 部分対応 | [アトミック](#atomics)を参照 |
atomic_max_acquire | 部分対応 | [アトミック](#atomics)を参照 |
atomic_max_acqrel | 部分対応 | [アトミック](#atomics)を参照 |
atomic_max_release | 部分対応 | [アトミック](#atomics)を参照 |
atomic_max_relaxed | 部分対応 | [アトミック](#atomics)を参照 |
atomic_min_seqcst | 部分対応 | [アトミック](#atomics)を参照 |
atomic_min_acquire | 部分対応 | [アトミック](#atomics)を参照 |
atomic_min_acqrel | 部分対応 | [アトミック](#atomics)を参照 |
atomic_min_release | 部分対応 | [アトミック](#atomics)を参照 |
atomic_min_relaxed | 部分対応 | [アトミック](#atomics)を参照 |
atomic_nand_seqcst | 部分対応 | [アトミック](#atomics)を参照 |
atomic_nand_acquire | 部分対応 | [アトミック](#atomics)を参照 |
atomic_nand_acqrel | 部分対応 | [アトミック](#atomics)を参照 |
atomic_nand_release | 部分対応 | [アトミック](#atomics)を参照 |
atomic_nand_relaxed | 部分対応 | [アトミック](#atomics)を参照 |
atomic_or_seqcst | 部分対応 | [アトミック](#atomics)を参照 |
atomic_or_acquire | 部分対応 | [アトミック](#atomics)を参照 |
atomic_or_acqrel | 部分対応 | [アトミック](#atomics)を参照 |
atomic_or_release | 部分対応 | [アトミック](#atomics)を参照 |
atomic_or_relaxed | 部分対応 | [アトミック](#atomics)を参照 |
atomic_singlethreadfence_seqcst | 部分対応 | [アトミック](#atomics)を参照 |
atomic_singlethreadfence_acquire | 部分対応 | [アトミック](#atomics)を参照 |
atomic_singlethreadfence_acqrel | 部分対応 | [アトミック](#atomics)を参照 |
atomic_singlethreadfence_release | 部分対応 | [アトミック](#atomics)を参照 |
atomic_store_seqcst | 部分対応 | [アトミック](#atomics)を参照 |
atomic_store_release | 部分対応 | [アトミック](#atomics)を参照 |
atomic_store_relaxed | 部分対応 | [アトミック](#atomics)を参照 |
atomic_store_unordered | 部分対応 | [アトミック](#atomics)を参照 |
atomic_umax_seqcst | 部分対応 | [アトミック](#atomics)を参照 |
atomic_umax_acquire | 部分対応 | [アトミック](#atomics)を参照 |
atomic_umax_acqrel | 部分対応 | [アトミック](#atomics)を参照 |
atomic_umax_release | 部分対応 | [アトミック](#atomics)を参照 |
atomic_umax_relaxed | 部分対応 | [アトミック](#atomics)を参照 |
atomic_umin_seqcst | 部分対応 | [アトミック](#atomics)を参照 |
atomic_umin_acquire | 部分対応 | [アトミック](#atomics)を参照 |
atomic_umin_acqrel | 部分対応 | [アトミック](#atomics)を参照 |
atomic_umin_release | 部分対応 | [アトミック](#atomics)を参照 |
atomic_umin_relaxed | 部分対応 | [アトミック](#atomics)を参照 |
atomic_xadd_seqcst | 部分対応 | [アトミック](#atomics)を参照 |
atomic_xadd_acquire | 部分対応 | [アトミック](#atomics)を参照 |
atomic_xadd_acqrel | 部分対応 | [アトミック](#atomics)を参照 |
atomic_xadd_release | 部分対応 | [アトミック](#atomics)を参照 |
atomic_xadd_relaxed | 部分対応 | [アトミック](#atomics)を参照 |
atomic_xchg_seqcst | 部分対応 | [アトミック](#atomics)を参照 |
atomic_xchg_acquire | 部分対応 | [アトミック](#atomics)を参照 |
atomic_xchg_acqrel | 部分対応 | [アトミック](#atomics)を参照 |
atomic_xchg_release | 部分対応 | [アトミック](#atomics)を参照 |
atomic_xchg_relaxed | 部分対応 | [アトミック](#atomics)を参照 |
atomic_xor_seqcst | 部分対応 | [アトミック](#atomics)を参照 |
atomic_xor_acquire | 部分対応 | [アトミック](#atomics)を参照 |
atomic_xor_acqrel | 部分対応 | [アトミック](#atomics)を参照 |
atomic_xor_release | 部分対応 | [アトミック](#atomics)を参照 |
atomic_xor_relaxed | 部分対応 | [アトミック](#atomics)を参照 |
atomic_xsub_seqcst | 部分対応 | [アトミック](#atomics)を参照 |
atomic_xsub_acquire | 部分対応 | [アトミック](#atomics)を参照 |
atomic_xsub_acqrel | 部分対応 | [アトミック](#atomics)を参照 |
atomic_xsub_release | 部分対応 | [アトミック](#atomics)を参照 |
atomic_xsub_relaxed | 部分対応 | [アトミック](#atomics)を参照 |
blackbox | 対応 | |
bitreverse | 対応 | |
breakpoint | 対応 | |
bswap | 対応 | |
caller_location | 非対応 | |
ceilf32 | 対応 | |
ceilf64 | 対応 | |
copy | 対応 | |
copy_nonoverlapping | 対応 | |
copysignf32 | 対応 | |
copysignf64 | 対応 | |
cosf32 | 部分対応 | 結果は過大近似されます。その方法は[このテスト](https://github.com/model-checking/kani/blob/main/tests/kani/Intrinsics/Math/Trigonometry/cosf32.rs)で説明しています |
cosf64 | 部分対応 | 結果は過大近似されます。その方法は[このテスト](https://github.com/model-checking/kani/blob/main/tests/kani/Intrinsics/Math/Trigonometry/cosf64.rs)で説明しています |
ctlz | 対応 | |
ctlz_nonzero | 対応 | |
ctpop | 対応 | |
cttz | 対応 | |
cttz_nonzero | 対応 | |
discriminant_value | 対応 | |
drop_in_place | 非対応 | |
exact_div | 対応 | |
exp2f32 | 部分対応 | 結果は過大近似されます |
exp2f64 | 部分対応 | 結果は過大近似されます |
expf32 | 部分対応 | 結果は過大近似されます |
expf64 | 部分対応 | 結果は過大近似されます |
fabs | 対応 | |
fadd_fast | 対応 | |
fdiv_fast | 部分対応 | [#809](https://github.com/model-checking/kani/issues/809) |
float_to_int_unchecked | 対応 | |
floorf32 | 対応 | |
floorf64 | 対応 | |
fmaf32 | 部分対応 | 結果は過大近似されます |
fmaf64 | 部分対応 | 結果は過大近似されます |
fmul_fast | 部分対応 | [#809](https://github.com/model-checking/kani/issues/809) |
forget | 対応 | |
frem_fast | 非対応 | |
fsub_fast | 対応 | |
likely | 対応 | |
log10f32 | 部分対応 | 結果は過大近似されます |
log10f64 | 部分対応 | 結果は過大近似されます |
log2f32 | 部分対応 | 結果は過大近似されます |
log2f64 | 部分対応 | 結果は過大近似されます |
logf32 | 部分対応 | 結果は過大近似されます |
logf64 | 部分対応 | 結果は過大近似されます |
maximum_number_nsz_f32 | 対応 | |
maximum_number_nsz_f64 | 対応 | |
align_of | 対応 | |
align_of_val | 対応 | |
minimum_number_nsz_f32 | 対応 | |
minimum_number_nsz_f64 | 対応 | |
move_val_init | 非対応 | |
mul_with_overflow | 対応 | |
needs_drop | 対応 | |
nontemporal_store | 非対応 | |
offset | 部分対応 | [すべての UB の条件](https://doc.rust-lang.org/std/primitive.pointer.html#safety-2)を検査しません |
powf32 | 部分対応 | 結果は過大近似されます |
powf64 | 部分対応 | 結果は過大近似されます |
powif32 | 部分対応 | 結果は過大近似されます |
powif64 | 部分対応 | 結果は過大近似されます |
prefetch_read_data | 非対応 | |
prefetch_read_instruction | 非対応 | |
prefetch_write_data | 非対応 | |
prefetch_write_instruction | 非対応 | |
ptr_guaranteed_eq | 対応 | |
ptr_guaranteed_ne | 対応 | |
ptr_offset_from | 部分対応 | [すべての UB の条件](https://doc.rust-lang.org/std/primitive.pointer.html#safety-4)を検査しません |
raw_eq | 部分対応 | [未初期化メモリ](#uninitialized-memory)を検出できません |
round_ties_even_f16 | 非対応 | |
round_ties_even_f32 | 対応 | |
round_ties_even_f64 | 対応 | |
round_ties_even_f128 | 非対応 | |
rotate_left | 対応 | |
rotate_right | 対応 | |
roundf32 | 対応 | |
roundf64 | 対応 | |
rustc_peek | 非対応 | |
saturating_add | 対応 | |
saturating_sub | 対応 | |
sinf32 | 部分対応 | 結果は過大近似されます。その方法は[このテスト](https://github.com/model-checking/kani/blob/main/tests/kani/Intrinsics/Math/Trigonometry/sinf32.rs)で説明しています |
sinf64 | 部分対応 | 結果は過大近似されます。その方法は[このテスト](https://github.com/model-checking/kani/blob/main/tests/kani/Intrinsics/Math/Trigonometry/sinf64.rs)で説明しています |
size_of | 対応 | |
size_of_val | 対応 | |
sqrtf32 | 部分対応 | 結果は過大近似されます |
sqrtf64 | 部分対応 | 結果は過大近似されます |
sub_with_overflow | 対応 | |
transmute | 部分対応 | [すべての UB の条件](https://doc.rust-lang.org/nomicon/transmutes.html)を検査しません |
truncf32 | 対応 | |
truncf64 | 対応 | |
try | 非対応 | [#267](https://github.com/model-checking/kani/issues/267) |
type_id | 対応 | |
type_name | 対応 | |
typed_swap_nonoverlapping | 対応 | |
unaligned_volatile_load | 部分対応 | [備考 - 並行性](#concurrency)を参照 |
unaligned_volatile_store | 部分対応 | [備考 - 並行性](#concurrency)を参照 |
unchecked_add | 対応 | |
unchecked_div | 対応 | |
unchecked_mul | 対応 | |
unchecked_rem | 対応 | |
unchecked_shl | 対応 | |
unchecked_shr | 対応 | |
unchecked_sub | 対応 | |
unlikely | 対応 | |
unreachable | 対応 | |
variant_count | 対応 | |
volatile_copy_memory | 部分対応 | [備考 - 並行性](#concurrency)を参照 |
volatile_copy_nonoverlapping_memory | 部分対応 | [備考 - 並行性](#concurrency)を参照 |
volatile_load | 部分対応 | [備考 - 並行性](#concurrency)を参照 |
volatile_set_memory | 部分対応 | [備考 - 並行性](#concurrency)を参照 |
volatile_store | 部分対応 | [備考 - 並行性](#concurrency)を参照 |
wrapping_add | 対応 | |
wrapping_mul | 対応 | |
wrapping_sub | 対応 | |
write_bytes | 対応 | |

#### アトミック {#atomics}

すべてのアトミックなイントリンシックは、その操作が行われるアトミックなブロックとして
コンパイルされます。ただし、[備考 - 並行性](#concurrency)で述べたように、Kani の
並行性の検証のサポートは限られており、デフォルトでは使われません。Kani はコードが
逐次的であると仮定するので、アトミックなイントリンシックを含むコードの検証は
信頼すべきではありません。

### プラットフォームのイントリンシック {#platform-intrinsics}

[`platform_intrinsics` フィーチャ](https://rust-lang.github.io/rfcs/1199-simd-infrastructure.html#operations)のイントリンシックです。

名前 | サポート | 備考 |
--- | --- | --- |
`simd_add` | 対応 | |
`simd_and`  | 対応 | |
`simd_div`  | 対応 | |
`simd_eq`  | 対応 | |
`simd_extract`  | 対応 | |
`simd_ge`  | 対応 | |
`simd_gt`  | 対応 | |
`simd_insert`  | 対応 | |
`simd_le`  | 対応 | |
`simd_lt`  | 対応 | |
`simd_mul`  | 対応 | |
`simd_ne`  | 対応 | |
`simd_or`  | 対応 | |
`simd_reduce_max`  | 対応 | 整数のレーン（このイントリンシックが受け付ける唯一のレーン） |
`simd_reduce_min`  | 対応 | 整数のレーン（このイントリンシックが受け付ける唯一のレーン） |
`simd_rem`  | 対応 | 浮動小数点数のオーバーフローを検査しません [#2669](https://github.com/model-checking/kani/issues/2669) |
`simd_shl`  | 対応 | |
`simd_shr`  | 対応 | |
`simd_shuffle*`  | 対応 | |
`simd_sub`  | 対応 | |
`simd_xor`  | 対応 | |
{% endraw %}
