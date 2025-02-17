How to hack TinyScheme
======================

TinyScheme is easy to learn and modify. It is structured like a
meta-interpreter, only it is written in C. All data are Scheme
objects, which facilitates both understanding/modifying the
code and reifying the interpreter workings.

In place of a dry description, we will pace through the addition
of a useful new datatype: garbage-collected memory blocks.
The interface will be:

* `(make-block <n> [<fill>])` makes a new block of the specified size
  optionally filling it with a specified byte
* `(block? <obj>)`
* `(block-length <block>)`
* `(block-ref <block> <index>)` retrieves byte at location
* `(block-set! <block> <index> <byte>)` modifies byte at location

In the sequel, lines that begin with '>' denote lines to add to the
code. Lines that begin with '|' are just citations of existing code.
Lines that begin with X denote lines to be removed from the code.

First of all, we need to assign a typeid to our new type. Typeids
in TinyScheme are small integers declared in the `scheme_types` enum
located near the top of the `scheme.c` file; it begins with `T_STRING`.
Add a new entry at the end, say `T_MEMBLOCK`. Remember to adjust the
value of `T_LAST_SYSTEM_TYPE` when adding new entries. There can be at
most 31 types, but you don't have to worry about that limit yet.

```
|       T_ENVIRONMENT=14,
X       T_LAST_SYSTEM_TYPE=14
>       T_MEMBLOCK=15,
>       T_LAST_SYSTEM_TYPE=15
|     };
```

Then, some helper macros would be useful. Go to where `is_string()`
and the rest are defined and add:
```
>     INTERFACE INLINE int is_memblock(pointer p)     { return (type(p)==T_MEMBLOCK); }
```

This actually is a function, because it is meant to be exported by
`scheme.h`. If no foreign function will ever manipulate a memory block,
you can instead define it as a macro:

```
>     #define is_memblock(p) (type(p)==T_MEMBLOCK)
```

Then we make space for the new type in the main data structure:
struct cell. As it happens, the `_string` part of the union `_object`
(that is used to hold character strings) has two fields that suit us:
```
|         struct {
|              char   *_svalue;
|              int   _keynum;
|         } _string;
```

We can use `_svalue` to hold the actual pointer and `_keynum` to hold its
length. If we couln't reuse existing fields, we could always add other
alternatives in union `_object`.

We then proceed to write the function that actually makes a new block.
For conformance reasons, we name it `mk_memblock`
```
>     static pointer mk_memblock(scheme *sc, int len, char fill) {
>          pointer x;
>          char *p=(char*)sc->malloc(len);
>
>          if(p==0) {
>               return sc->NIL;
>          }
>          x = get_cell(sc, sc->NIL, sc->NIL);
>
>          typeflag(x) = T_MEMBLOCK|T_ATOM;
>          strvalue(x)=p;
>          keynum(x)=len;
>          memset(p,fill,len);
>          return (x);
>     }
```

The memory used by the MEMBLOCK will have to be freed when the cell
is reclaimed during garbage collection. There is a placeholder for
that staff, function `finalize_cell()`, currently handling strings only.
```
|     static void finalize_cell(scheme *sc, pointer a) {
|       if(is_string(a)) {
|          sc->free(strvalue(a));
>       } else if(is_memblock(a)) {
>          sc->free(strvalue(a));
|       } else if(is_port(a)) {
```

There are no MEMBLOCK literals, so we don't concern ourselves with
the READER part (yet!). We must cater to the PRINTER, though. We
add one case more in `atom2str()`.
```
|     } else if (iscontinuation(l)) {
|          p = "#<CONTINUATION>";
>     } else if (is_memblock(l)) {
>          p = "#<MEMORY BLOCK>";
|     } else {
```

Whenever a MEMBLOCK is displayed, it will look like that.
Now, we must add the interface functions: constructor, predicate,
accessor, modifier. We must in fact create new op-codes for the virtual
machine underlying TinyScheme. Since version 1.30, TinyScheme uses
macros and a single source text to keep the enums and the dispatch table
in sync. The op-codes are defined in the opdefines.h file with one line
for each op-code. The lines in the file have six columns between the
starting `_OPDEF(` and ending `)`: A, B, C, D, E, and OP.
Note that this file uses unusually long lines to accomodate all the
information; adjust your editor to handle this.

The purpose of the columns is:

  - Column A is the name of the subroutine that handles the op-code.
  - Column B is the name of the op-code function.
  - Columns C and D are the minimum and maximum number of arguments
    that are accepted by the op-code.
  - Column E is a set of flags that tells the interpreter the type of
    each of the arguments expected by the op-code.
  - Column OP is used in the scheme_opcodes enum located in the
    `scheme-private.h` file.

Op-codes are really just tags for a huge C switch, only this switch
is broken up in to a number of different `opexe_X` functions. The
correspondence is made in table "`dispatch_table`". There, we assign
the new op-codes to `opexe_2`, where the equivalent ones for vectors
are situated. We also assign a name for them, and specify the minimum
and maximum arity (number of expected arguments). `INF_ARG` as a maximum
arity means "unlimited".

For reasons of consistency, we add the new op-codes right after those
for vectors:
```
|     _OP_DEF(opexe_2, "vector-set!",                    3,  3,       TST_VECTOR TST_NATURAL TST_ANY,  OP_VECSET           )
>     _OP_DEF(opexe_2, "make-block",                     1,  2,       TST_NATURAL TST_CHAR,            OP_MKBLOCK          )
>     _OP_DEF(opexe_2, "block-length",                   1,  1,       T_MEMBLOCK,                      OP_BLOCKLEN         )
>     _OP_DEF(opexe_2, "block-ref",                      2,  2,       T_MEMBLOCK TST_NATURAL,          OP_BLOCKREF         )
>     _OP_DEF(opexe_2, "block-set!",                     1,  1,       T_MEMBLOCK TST_NATURAL TST_CHAR, OP_BLOCKSET         )
|     _OP_DEF(opexe_3, "not",                            1,  1,       TST_NONE,                        OP_NOT              )
```

We add the predicate along with the other predicates in `opexe_3`:
```
|     _OP_DEF(opexe_3, "vector?",                        1,  1,       TST_ANY,                         OP_VECTORP          )
>     _OP_DEF(opexe_3, "block?",                         1,  1,       TST_ANY,                         OP_BLOCKP           )
|     _OP_DEF(opexe_3, "eq?",                            2,  2,       TST_ANY,                         OP_EQ               )
```

All that remains is to write the actual code to do the processing and
add it to the switch statement in `opexe_2`, after the `OP_VECSET` case.
```
>     case OP_MKBLOCK: { /* make-block */
>          int fill=0;
>          int len;
>
>          if(!isnumber(car(sc->args))) {
>               Error_1(sc,"make-block: not a number:",car(sc->args));
>          }
>          len=ivalue(car(sc->args));
>          if(len<=0) {
>               Error_1(sc,"make-block: not positive:",car(sc->args));
>          }
>
>          if(cdr(sc->args)!=sc->NIL) {
>               if(!isnumber(cadr(sc->args)) || ivalue(cadr(sc->args))<0) {
>                    Error_1(sc,"make-block: not a positive number:",cadr(sc->args));
>               }
>               fill=charvalue(cadr(sc->args))%255;
>          }
>          s_return(sc,mk_memblock(sc,len,(char)fill));
>     }
>
>     case OP_BLOCKLEN:  /* block-length */
>          if(!ismemblock(car(sc->args))) {
>               Error_1(sc,"block-length: not a memory block:",car(sc->args));
>          }
>          s_return(sc,mk_integer(sc,keynum(car(sc->args))));
>
>     case OP_BLOCKREF: { /* block-ref */
>          char *str;
>          int index;
>
>          if(!ismemblock(car(sc->args))) {
>               Error_1(sc,"block-ref: not a memory block:",car(sc->args));
>          }
>          str=strvalue(car(sc->args));
>
>          if(cdr(sc->args)==sc->NIL) {
>               Error_0(sc,"block-ref: needs two arguments");
>          }
>          if(!isnumber(cadr(sc->args))) {
>               Error_1(sc,"block-ref: not a number:",cadr(sc->args));
>          }
>          index=ivalue(cadr(sc->args));
>
>          if(index<0 || index>=keynum(car(sc->args))) {
>               Error_1(sc,"block-ref: out of bounds:",cadr(sc->args));
>          }
>
>          s_return(sc,mk_integer(sc,str[index]));
>     }
>
>     case OP_BLOCKSET: { /* block-set! */
>          char *str;
>          int index;
>          int c;
>
>          if(!ismemblock(car(sc->args))) {
>               Error_1(sc,"block-set!: not a memory block:",car(sc->args));
>          }
>          if(isimmutable(car(sc->args))) {
>               Error_1(sc,"block-set!: unable to alter immutable memory block:",car(sc->args));
>          }
>          str=strvalue(car(sc->args));
>
>          if(cdr(sc->args)==sc->NIL) {
>               Error_0(sc,"block-set!: needs three arguments");
>          }
>          if(!isnumber(cadr(sc->args))) {
>               Error_1(sc,"block-set!: not a number:",cadr(sc->args));
>          }
>          index=ivalue(cadr(sc->args));
>          if(index<0 || index>=keynum(car(sc->args))) {
>               Error_1(sc,"block-set!: out of bounds:",cadr(sc->args));
>          }
>
>          if(cddr(sc->args)==sc->NIL) {
>               Error_0(sc,"block-set!: needs three arguments");
>          }
>          if(!isinteger(caddr(sc->args))) {
>               Error_1(sc,"block-set!: not an integer:",caddr(sc->args));
>          }
>          c=ivalue(caddr(sc->args))%255;
>
>          str[index]=(char)c;
>          s_return(sc,car(sc->args));
>     }
```

Finally, do the same for the predicate in `opexe_3`.
```
|     case OP_VECTORP:     /* vector? */
|          s_retbool(is_vector(car(sc->args)));
>     case OP_BLOCKP:     /* block? */
>          s_retbool(is_memblock(car(sc->args)));
|     case OP_EQ:         /* eq? */
```
