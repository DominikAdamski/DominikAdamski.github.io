---
layout: post
title: "Abstract syntax trees in GCC and in Clang"
author: "Dominik Adamski"
---


GCC and Clang are two major open source compilers. They offer comparable code optimization opportunities, but their internal design differs. Let's analyze implementation of GCC 8 and Clang 12 frontends to see how different assumptions and high level design decisions make an impact on compilers' implementation.

## Code organization

GCC supports multiple input languages not only from C-family. It can also compile code written in languages like Fortran or Go. Each supported language has its dedicated frontend which converts input source code into common intermediate representation. The source code of each frontend can be found under the gcc_root/gcc/name_of_language directory. The frontends can have internal dependencies. For example, ada frontend is dependent on C and C++ frontend. The dependencies are described by [config-lang.in file](https://gcc.gnu.org/onlinedocs/gccint/Front-End-Config.html#Front-End-Config) which is required to be specified for each frontend. Another obligatory file for each frontend is [Make-lang.in file](https://gcc.gnu.org/onlinedocs/gccint/Front-End-Makefile.html#Front-End-Makefile). These files are part of the GCC build system and they define how each frontend should be built. Detailed information [how to add support for new language is placed in GCC documentation](https://gcc.gnu.org/onlinedocs/gccint/Front-End.html#Front-End).

Clang is frontend for LLVM optimizer. In comparison to GCC it is developed only as a frontend for C-family languages. Clang like LLVM can be used not only as a standalone compiler but it can be also treated as a set of libraries which can be used separately. That's why the source code is organized differently. It is divided into libraries. Each library is placed into a separate directory and it is responsible for different parts of the compilation process. The interfaces between libraries are well defined and it is easy to use the existing source code to create new tools (for example for code formatting like clang-format, clang-check etc.). The source code of the clang libraries can be found under the llvm_root/clang/lib directory. Each subdirectory contains source code of one library (for example Lex subdirectory contains source code for lexical analysis). [Clang provides extensive documentation](https://clang.llvm.org/docs/Tooling.html) how to use its source code to create new tools which support developers.

## AST

GCC and Clang make extensive use of Abstract Syntax Tree (AST) objects. Compilers create AST objects which reflect input source code before code optimization. ASTs are useful for providing detailed information about errors in the compiled source code. They not only improve error diagnostics, but they also can be used in code refactoring tools. ASTs are an important part of the compiler frontend because they not only reflect input source code but they also ease code lowering into intermediate code.

### ASTs in GCC

Each GCC frontend can implement its own representation of AST. GCC requires that custom ASTs are converted into Generic ASTs which are common representations for all languages. Generic ASTs are used as the input for optimization procedure which is common for multiple compiler frontends and backends whereas custom ASTs better reflect language specific features (like lambda functions for C++). Frontend specific ASTs enable more meaningful error diagnostics for developers in case of errors in compiled source codes.

GCC frontend for C/C++ relies on generic ASTs extended by language specific ASTs. Generic ASTs are implemented as a set of connected `tree_node` objects. `tree_node` is defined as a union of structures. The definition of `tree_node` union and its structures is available in `gcc_root/gcc/tree-core.h` header file:

```c
union GTY ((ptr_alias (union lang_tree_node),
        desc ("tree_node_structure (&%h)"), variable_size)) tree_node {
  struct tree_base GTY ((tag ("TS_BASE"))) base;
  struct tree_typed GTY ((tag ("TS_TYPED"))) typed;
  struct tree_common GTY ((tag ("TS_COMMON"))) common;
  struct tree_int_cst GTY ((tag ("TS_INT_CST"))) int_cst;
  struct tree_poly_int_cst GTY ((tag ("TS_POLY_INT_CST"))) poly_int_cst;
  struct tree_real_cst GTY ((tag ("TS_REAL_CST"))) real_cst;
  struct tree_fixed_cst GTY ((tag ("TS_FIXED_CST"))) fixed_cst;
  struct tree_vector GTY ((tag ("TS_VECTOR"))) vector;
  struct tree_string GTY ((tag ("TS_STRING"))) string;
  struct tree_complex GTY ((tag ("TS_COMPLEX"))) complex;
  struct tree_identifier GTY ((tag ("TS_IDENTIFIER"))) identifier;
  struct tree_decl_minimal GTY((tag ("TS_DECL_MINIMAL"))) decl_minimal;
  struct tree_decl_common GTY ((tag ("TS_DECL_COMMON"))) decl_common;
  struct tree_decl_with_rtl GTY ((tag ("TS_DECL_WRTL"))) decl_with_rtl;
  struct tree_decl_non_common  GTY ((tag ("TS_DECL_NON_COMMON")))
    decl_non_common;
  struct tree_parm_decl  GTY  ((tag ("TS_PARM_DECL"))) parm_decl;
  struct tree_decl_with_vis GTY ((tag ("TS_DECL_WITH_VIS"))) decl_with_vis;
  struct tree_var_decl GTY ((tag ("TS_VAR_DECL"))) var_decl;
  struct tree_field_decl GTY ((tag ("TS_FIELD_DECL"))) field_decl;
  struct tree_label_decl GTY ((tag ("TS_LABEL_DECL"))) label_decl;
  struct tree_result_decl GTY ((tag ("TS_RESULT_DECL"))) result_decl;
  struct tree_const_decl GTY ((tag ("TS_CONST_DECL"))) const_decl;
  struct tree_type_decl GTY ((tag ("TS_TYPE_DECL"))) type_decl;
  struct tree_function_decl GTY ((tag ("TS_FUNCTION_DECL"))) function_decl;
  struct tree_translation_unit_decl GTY ((tag ("TS_TRANSLATION_UNIT_DECL")))
    translation_unit_decl;
  struct tree_type_common GTY ((tag ("TS_TYPE_COMMON"))) type_common;
  struct tree_type_with_lang_specific GTY ((tag ("TS_TYPE_WITH_LANG_SPECIFIC")))
    type_with_lang_specific;
  struct tree_type_non_common GTY ((tag ("TS_TYPE_NON_COMMON")))
    type_non_common;
  struct tree_list GTY ((tag ("TS_LIST"))) list;
  struct tree_vec GTY ((tag ("TS_VEC"))) vec;
  struct tree_exp GTY ((tag ("TS_EXP"))) exp;
  struct tree_ssa_name GTY ((tag ("TS_SSA_NAME"))) ssa_name;
  struct tree_block GTY ((tag ("TS_BLOCK"))) block;
  struct tree_binfo GTY ((tag ("TS_BINFO"))) binfo;
  struct tree_statement_list GTY ((tag ("TS_STATEMENT_LIST"))) stmt_list;
  struct tree_constructor GTY ((tag ("TS_CONSTRUCTOR"))) constructor;
  struct tree_omp_clause GTY ((tag ("TS_OMP_CLAUSE"))) omp_clause;
  struct tree_optimization_option GTY ((tag ("TS_OPTIMIZATION"))) optimization;
  struct tree_target_option GTY ((tag ("TS_TARGET_OPTION"))) target_option;
};
```

Each structure of `tree_node` describes a different part of source code. There are structures which describe declarations of variables (for example `struct tree_function_decl`), OpenMP clauses (`struct tree_omp_clause`) or list of parsed statements (`struct tree_statement_list`). `tree_node` union should be treated as an abstract interface to AST objects. For convenience `tree` alias is introduced with the pointer type `union tree_node*` in `gcc_root/gcc/coretypes.h` file:

```c
typedef union tree_node *tree;
typedef const union tree_node *const_tree;
```

#### Handling of GCC's trees

Implementation of `tree_node` as a union of structures is problematic because C does not provide a mechanism of run-time type information. In consequence GCC developers are required to use special macros which allow to indicate the exact type of passed `tree`. `gcc_root/gcc/tree.h` file contains definitions of macros for generic ASTs. For example, the macro `TREE_CODE` returns the type id of passed tree. Information about type id is stored inside the `tree_base` structure. This structure is part of all other structures mentioned in `tree_node` union and it stores information about each tree such as: information about side effects of the source code mapped into that tree, language specific flags attached to compiled source code or data alignment. Definition of the `tree_base` structure is available in the `gcc_root/gcc/tree-core.h` file. Information about type id is represented by `enum tree_code` defined in `tree-core.h` header:

```c
#define DEFTREECODE(SYM, STRING, TYPE, NARGS)   SYM,
#define END_OF_BASE_TREE_CODES LAST_AND_UNUSED_TREE_CODE,

enum tree_code {
#include "all-tree.def"
MAX_TREE_CODES
};

#undef DEFTREECODE
#undef END_OF_BASE_TREE_CODES
```

This enum is created during building of gcc project. The content of this enumeration is dependent on number of languages supported by gcc. Makefie recipes create `all-tree.def` before gcc's source files are compiled. If given language is supported, then they attach language specific tree definition file into generated `all-tree.def` file. Exact procedure of creating `all-tree.def` is defined in `gcc_root/gcc/Makefile.in`:

```bash
# all-tree.def includes all the tree.def files.
all-tree.def: s-alltree; @true
s-alltree: Makefile
    rm -f tmp-all-tree.def
    echo '#include "tree.def"' > tmp-all-tree.def
    echo 'END_OF_BASE_TREE_CODES' >> tmp-all-tree.def
    echo '#include "c-family/c-common.def"' >> tmp-all-tree.def
    ltf="$(lang_tree_files)"; for f in $$ltf; do \
      echo "#include \"$$f\""; \
    done | sed 's|$(srcdir)/||' >> tmp-all-tree.def
    $(SHELL) $(srcdir)/../move-if-change tmp-all-tree.def all-tree.def
    $(STAMP) s-alltree
```

Definition file contains not only information about raw typeid, but also additional data which precisely describes given tree type. Each type is defined as DEFTREECODE macro with four arguments. The first argument denotes type id, the second is the name (string) of the type id, the third argument describes associated tree code class id and the fourth argument gives information about additional arguments required for a given tree. GCC contains multiple implementations of DEFTREECODE macro. Each implementation returns only data which is currently required (for example DEFTREECODE macro in `tree-core.h` file returns only typeid which is required to create enumeration list).

Another macro which is useful in development of new features is `TREE_CHECK`. It checks if the passed `tree` is accessed correctly. In case of failure it generates internal error. It allows to eliminate memory access violation caused by incorrect handling of `tree` pointers which can point to different structures. GCC contains plenty of useful macros for inspecting ASTs. Some of these macros are frontend specific and they are placed under the frontend directory. (For example C++ AST specific macros can be found in `gcc_root/gcc/cp/cp-tree.h` file). GCC also generates some standard macros for AST checking during the build process. They can be found in the `tree-check.h` file under `host_build` directory.

### ASTs in Clang

Clang is designed in conformance to rules for object oriented programming. Classes which describe ASTs are grouped into libAST library. These classes try to reflect input source code as much as possible.

 
The base class for all subtypes of ASTs is called Stmt. This class acts as an interface to classes which reflects ASTs of language constructs like if stmts, do/while loops, etc. Every element of C-like language grammar is modelled by dedicated class. Inheritance diagram for `Stmt` class shows that there are classes which are dedicated only for ObjectiveC (their names starts with ObjC prefix), or C++ (prefix CXX). It's worth to mention that there are seperate classes to model classic for loop and C++ for-range loop even though their meaning is similar.

![Clang Stmt inheritance diagram](https://raw.githubusercontent.com/DominikAdamski/DominikAdamski.github.io/master/assets/clang_stmt_inheritance_graph.png "Clang Stmt inheritance diagram")
_Partial list of classes which are derived from clang::Stmt class_

Clang AST library requires that each subclass of `Stmt` class contains `child_terator` for easy graph traversal. The meaning of children can vary across AST subclasses. For example, children of `CompoundStmt` are other statements listed in the source code within `{ }`brackets, whereas children of `IfStmt` are statements connected with condition expression, then statement and optional else statement.

In comparison to GCC, there is no need to use special type ids and dedicated macros, to differentiate between different types of AST statements. Statement class derived from Clang's `Stmt` class contains a static function `classof` which returns true if a given object belongs to that class. Implementation of AST as C++ classes in Clang makes memory management easier as for GCC. The GCC project uses special macros to correctly free allocated memory, whereas Clang applies standard C++ memory handling methods.

## Documentation

This article only scratches the surface of AST in compiler development. More sophisticated high level descriptions can be found inside project documentation.

Clang provides extensive developer documentation. Newbies can learn more about AST [by watching presentations of Clang internals](https://clang.llvm.org/docs/IntroductionToTheClangAST.html). The high level documentation [can be found on project website](https://clang.llvm.org/docs/InternalsManual.html). Clang also provides [doxygen documentation for the recent version of the code](https://clang.llvm.org/doxygen/). The documentation for older releases can be generated by the developer as a Makefile target.


GCC [provides document](https://gcc.gnu.org/onlinedocs/gccint.pdf) which briefly describes design concepts of GCC.

Last but not least the code is the best documentation. You can get the code and explore it by yourself.

## Summary

Both compilers implement the same concept of ASTs but they do it in different ways. GCC sticks to the C interface (pointer to the union) and they aim to cover as many input languages as possible. GCC developers provide solutions which perfectly fit their codebase ([which was written only in C from 1987 till 2010](https://www.developer.com/news/allowing-c-in-gcc-the-gnu-compiler-collection/) ). Whereas Clang provides a modular, object oriented libraries for C-family languages which can be easily incorporated to commercial tools due to permissive license (Apache 2.0 License vs GPL/LGPL). 


## Further reading

I add some links to useful articles which put more light on compiler internals

* [LWN article about LLVM and GCC internals](https://lwn.net/Articles/582697/) 
* [Stephan’s Friedl article about GCC ASTs](https://stephanfr.com/tag/gcc-ast/)
* Last but not least, great theoretical introduction: the dragon book - [Aho, Sethi, Ullman, Compilers: Principles, Techniques, and Tools ](https://en.wikipedia.org/wiki/Compilers:_Principles,_Techniques,_and_Tools)


