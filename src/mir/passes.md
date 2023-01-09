# MIR passes: getting the MIR for a function
## MIR通行证：Mir passes
如果希望获取函数（或常量等）的 MIR，可以使用 optimized_MIR（def_id）查询。这将为您返回最终优化的MIR。对于外部def-id，我们只需从其他板条箱（crate）的元数据中读取MIR。但是对于本地def-id，查询将构造MIR，然后通过应用一系列过程来迭代优化它。本节介绍这些传递的工作原理以及如何扩展它们。


为了为给定的def-id D生成optimized_mir（D），mir需要经过多个优化套件的处理，每个优化都由一个查询表示。每个套件都包含多个优化和转换。这些套件代表了我们希望访问MIR以进行类型检查或其他目的的有用中间点：

mir_build（D）–不是查询，但它构造了初始mir

mir_const（D）–应用一些简单的变换，使mir准备好进行常量求值；

mir_validated（D）–应用更多转换，使mir准备好进行借用检查；

optimized_mir（D）–执行所有优化后的最终状态。


## 实现与注册Pass

MirPass是一些处理 MIR 的代码，通常（但并非总是）以某种方式对其进行转换。例如，它可能会执行优化。MirPass特征定义在[`rustc_mir_transform crate`]，它基本上由一个方法 run_pass 组成，该方法只需获取一个&mut mir（以及tcx和一些关于它来自何处的信息）。因此，MIR是在某个 place 被修改的（这有助于保持效率）。

MIR过程的一个基本示例是[`RemoveStorageMarkers`]，它遍历 MIR 并删除所有存储标记（如果在代码生成期间将不会发出）。从其源代码中可以看到，MIR过程是通过首先定义一个伪类型（一个没有字段的结构）来定义的，比如：

    struct MyPass；
然后为 MyPass 实现 MirPass 特性。您接下来可以将此pass插入到在查询中找到的合适的 passes 列表中，如 optimized_mir、mir_validated 等（如果这是一个优化，则应将其插入 optimized_mir 列表中。）

如果你正在写pass，这是一个很好的使用[`MIR visitor`]的机会。MIR visitor是一种方便的方式，可以经历MIR的各个环节，无论是搜索还是进行小的编辑。



## Stealing-窃取

中间查询 mir_const() 和 mir_validated() 产生一个使用 tcx.alloc_stal_mir() 分配，生命周期为 &'tcx的 Steal<Mir<'tcx>>。这表明结果可能被下一组优化窃取——这是一个避免克隆 mir 的优化。除非作为 MIR 处理管道的一部分，否则不要直接从这些中间查询中读取数据，试图使用窃取的结果将导致编译器中的panic。

由于这种窃取机制，还必须注意确保在处理管道中特定阶段的MIR被窃取之前，任何可能想要从中读取的人都已经这样做了。具体地说，这意味着如果您有一些查询 foo(D)想要访问 MIR_const(D) 或 MIR_validated(D) 的结果，您需让后继者使用 ty::queries::foo::force(...) 传递 “force”foo(D)。这将强制执行查询，即使您不直接要求其结果。

以 MIR 常量限定为例。它想读取 mir_const() 套件产生的结果。然而，该结果将被mir_validated()套件窃取。如果未执行任何操作，则 mir_const_qualif(D) 将在 mir_validated(D) 之前成功，否则将失败。因此， mir_validated(D) 将在实际窃取之前强制执行 mir_const_qualif ，从而确保读取已经发生（记住[`查询被存储`][queries_memoized]，因此执行两次查询，第二次只需从缓存加载）：

mir_const(D) --read-by--> mir_const_qualif(D)

     |                       ^

  stolen-by                  |

     |                    (forces)

     v                       |

mir_validated(D) ------------+


这个机制还有一点瑕疵。[`rust-lang/rust#41710`]中讨论了更优雅的替代方案。

[`rustc_mir_transform crate`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_mir_transform/
[`RemoveStorageMarkers`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_mir_transform/remove_storage_markers/struct.RemoveStorageMarkers.html
[`MIR visitor`]: https://rustc-dev-guide.rust-lang.org/mir/visitor.html
[queries_memoized]: https://rustc-dev-guide.rust-lang.org/query.html
[`rust-lang/rust#41710`]: https://github.com/rust-lang/rust/issues/41710
