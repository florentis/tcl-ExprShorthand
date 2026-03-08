# TIP :           xxx
     Title:	  Mathematical expression substitution 
     Version:  $Revision: 0.2 $
     Author:  Florent MERLET <florent.merlet.auditeur@lecnam.net>
     State:  Draft
     Type:  Project
     Tcl-Version: 9.1
     Vote:   Pending
     Created: 2026-03-01
     Post-History:
------

## Abstract
  This TIP proposes to extend the Tcl syntax, in including a new rule of substitution :   
  ***mathematical expression substitution***. 

## Rationale
Improving the readability of arithmetic calculations is a long term demand in Tcl. One of this first demand was [TIP 282](https://core.tcl-lang.org/tips/doc/trunk/tip/282.md). Periodically, expansive discussions about it occur on the wiki. There were many suggestions to make things more practical : [TIP 676](https://core.tcl-lang.org/tips/doc/trunk/tip/676.md), [TIP 674](https://core.tcl-lang.org/tips/doc/trunk/tip/674.md), [TIP 672](https://core.tcl-lang.org/tips/doc/trunk/tip/672.md).

## Specification
At the Tcl parser level, the shorthand will allow this syntax :

```tcl
        set A [( *expression* )]
```

This corresponds to this new rule of substitution :
> * **Mathematical expression substitution** : If the first caracter of a word is an open-bracket and is immediately followed by an open-parenthese, then Tcl performs a *Mathematical expression substitution*. The expression has to follows the rules of the *expr* language. It must be closed by a closed-parenthese immediately followed by a closed-bracket. 

## Options
- **Shorthand in array indexes** : In the context of an array variable index, the shorthand will allow this syntax :

```tcl
         set A(( *expression* )) 1
```

This will create in the array 'A' a key whose value will be the result of the computed expression.

- **Native list handling** :

```tcl
         set m [((1, 0, 0), (0, 1, 0), (0, 0, 1))]
```

will be made equivalent to :

```tcl
          set m [list [list 1 0 0] [list 0 1 0] [list 0 0 1]]]
```

- Inclusion of **[TIP 282](https://core.tcl-lang.org/tips/doc/trunk/tip/282.md.html) proposals** :
  
And a little improvement to allow us to use a bareword for variable name on the left side of the assignement operator. 

Then, we can write :

```tcl
         set L [( x = 1; ($x, $x*2, $x*3) )]
		 # set x to 1 and set L to {1 2 3}
```

- **Scripted expressions** :

  Every value which begins with a `(` and finishes by a `)` will be evaluated, or compiled, as an expression when taken as a script by a command.

  ```tcl
  eval {(1+1)}; # return 2
  ```
  It should work with `if`, `while`, `for`, `foreach`, `lmap`, `eval`, `apply`, `proc`,...etc

## Examples :
- Setting a variable :
```tcl
set bright [($red*0.3 + $green*0.59 + $blue*0.11)]
```

- A cross product proc **with native list handling**
```tcl
    proc crossProduct {U V} {
       lassign $U x y z
       lassign $V u v w
       return [( $y*$w - $v*$z, $w*$x - $u*$z, $x*$v - $u*$y)]		
    }
```
- Eratosthenes sieve :
  - With **basic shorthand** :
  ```tcl
     proc sieve {n} {
       set L [lseq $n]
       for {set i 2} {$i < $n} {incr i} {
          set j $i
          while {$i*$j < $n} {
             lset L [( $i*$j )] {}
             incr j
          }
       }
       return [lsearch -all -not -exact $L {}]
    }
   ```

   - with **TIP 282** integration (illustrating with `for` and `while`):
   ```tcl
     proc sieve {n} {
        set L [lseq $n]
        for {set i 2} {(j = $i) < $n} {incr i} {
            while {(k = $i*$j) < $n} {
	           lset L $k {}
	           incr j
            }
       }
       return [lsearch -all -not -exact $L {}]
    }
   ```

- Matrix calculus and affine transform example **with native list handling** and **TIP 282**

```tcl

proc MatrixTranspose {M} {
    if {[llength [lindex $M 0]] == 1} {
        # M is a vector
        return $M
    }
    set i 1
    foreach row $M {
        lassign $row m${i}1 m${i}2 m${i}3
        incr i
    }

    return  [( $i == 2 ?
               (# it was a false matrix case : there was just one row after all.
                $m11, $m12, $m13 
               ) : (
                ($m11, $m21, $m31),
                ($m12, $m22, $m32),
                ($m13, $m23, $m33)
                )  )]
}

proc MatrixProduct {M1 M2} {
    set i 0
    foreach row $M1 {
        lassign $row m${i}0 m${i}1 m${i}2
        incr i
    }
        
    set R []
    foreach v [MatrixTranspose $M2] {
         lassign $v x y z
         lappend R [( [llength $v] == 1 ?
                      (vector = 1; # M2 is a vector !
                       y = [lindex $M2 1];
                       z = [lindex $M2 2];
                       ($m00*$x + $m01*$y + $m02*$z,
                        $m10*$x + $m11*$y + $m12*$z,
                        $m20*$x + $m21*$y + $m22*$z)
                       ) :
                      (vector = 0; # M2 was a matrix.
                      ($m00*$x + $m01*$y + $m02*$z,
                       $m10*$x + $m11*$y + $m12*$z,
                       $m20*$x + $m21*$y + $m22*$z)
                       ))]
         if {$vector == 1} {
               return {*}$R
         }
    }
    return {*}$R
}

# Affine transforms examples :

proc translation {dx dy} {
    return [( 	(1, 0, $dx),
              	(0, 1, $dy),
              	(0, 0,  1 )    )]
}

proc rotation {angle} {
    return [(  angle = $angle/180.0*acos(-1);
				(cos($angle), -sin($angle), 0),
               	(sin($angle),  cos($angle), 0),
               	(     0     ,      0      , 1)       )]
}

set Point  [(	100*cos(30.0/180*acos(-1)),
				50,
				1 )]

set RotatedPoint [MatrixProduct [rotation -30] $Point]
set TranslatedPoint [MatrixProduct [translation -100 0] $RotatedPoint]

puts RotatedPoint\ :\ $RotatedPoint
puts TranslatedPoint\ :\ $TranslatedPoint

# RotatedPoint : {100.00000000000001 7.105427357601002e-15 1.0}
# TranslatedPoint : {1.4210854715202004e-14 7.105427357601002e-15 1.0}
```
- Draw a rectangle on a canvas with **native list handling** 

```tcl
    .c create rect [($x, $y, $x+100, $y+100)]
```

- Tensorial product, with **native list handling** and **TIP 282**

```tcl
     proc TensorialProduct {V U} {
       lassign $V x y z
       lassign $U u v w

       return [( a11 = $x*$u;  a12 = $x*$v;  a13 = $x*$w;
	             a21 = $y*$u;  a22 = $y*$v;  a23 = $y*$w;
	             a31 = $z*$u;  a32 = $z*$v;  a33 = $z*$w;
	             ($a11, $a12, $a13),
	             ($a21, $a22, $a23),
	             ($a31, $a32, $a33)
		)]
     }
     puts [TensorialProduct {1 2 3} {3 2 1}]
     # {3 2 1} {6 4 2} {9 6 3}
```

- Matrix transpose (2) : with **main shorthand**, **native list handling**, **TIP 282**, **scripted expression**

```tcl
proc transpose {M} {
    set MAP {{a b c} {d e f} {g h i}}
    lmap row1 $MAP row2 $M {
    	lmap var $row1 val $row2 {($var = double($val))}
    }
    return [( 	($a, $d, $g),
	        	($b, $e, $h),
	      		($c, $f, $i) )]
}

```
## Implementation
The code is made on top of Tcl9.1a, in separated repositories, one for the main expr shorhand, the other one for the other optional features. Files tclsh.exe are compiled under cygwin above Win10 with gcc.

*Do note that all exemples written in the previous section have been tested and are working in the [tcl-ExprShorthand-index-list-TIP282](https://github.com/florentis/tcl-ExprShorthand-index-list-TIP282) implementation.*

### Code for main shorthand
It can be found at [tcl-ExprShorthand](https://github.com/florentis/tcl-ExprShorthand). It should allow this syntax :

     set A [(1+1)]; # $A is 2

#### Principles of the changes
 * In tclParse.c :
     * in *parseToken* : Add a test for the existence of an open parenthese after the open bracket, using the Parse_Expr parser, to get to the length of the expression. Mark this area as a TCL_TOKEN_SUB_EXPR.
     * in *substToken* : Add a case to check for a TCL_TOKEN_SUB_EXPR and call Tcl_ExprObj to substitute the token value.
 * In tclCompExpr.c :
   * in *ParseExpr* : Add a new variable substExpressionContext, some code to be able to nest the shorthand into the shorthand, and to control the end of the expression
   * in *Tcl_ParseEpr* : Manage the return of the length of the expression to *parseToken*
 * In TclCompile.c
     * in *TclCompileTokens* : Add a case to handle TCL_TOKEN_SUB_EXPR
     
### Code for array index shorthand
It can be found at [tcl-ExprShorthand-index](https://github.com/florentis/tcl-ExprShorthand-index). It includes the changes made for the main shorthand. It should allow this syntax :

      set B((1+1)) [(2+2)]; # $B(2) is 4

#### Principles of the changes

 * in TclVar.c
   * in *TclLookupArrayElement* : Check if the name of the element begin with a '(' and finish by ')'. If so, substitute its value by the expression value.
 * in TclCompile.c
   * in *TclPushVarName* :  Check if the name of the element begin with a '(' and finish by ')'. If so, mark it as TCL_TOKEN_SUB_EXPR.

### Code for native list handling
It can be found at [tcl-ExprShorthand-index-list](https://github.com/florentis/tcl-ExprShorthand-index-list). It includes the changes made for the main shorthand, as well as the changes made for the array index shorthand. It should allow this syntax :

     set C((1+1)) [(  (1+1, 2+2, 3+3),  
                      (2*4, 2*5, 2*6)    )]  
     # $C(2) is {{2 4 6} {8 10 12}}

#### Principles of the changes

  * In TclCompExpr.c :
     * In *ParseExpr* : In the case of OPEN_PAREN unary operator not preceded by a FUNCTION operator, create a FUNCTION operator and add a list function in the litlist. This allows to avoid the "comma out function argument" error and to return a list instead.
  * In tclBasic.c : add Tcl_ListObjCmd in the table for mathfunc.

### Code for TIP282 integration 
It can be found at [tcl-ExprShorthand-index-list-TIP282](https://github.com/florentis/tcl-ExprShorthand-index-list-TIP282). It includes all the previous additions, as well as the changes made to integrate TIP282. It should allow this syntax :

      set K [( x=10; y=50; ($x*$y, $y/$x) )]
      # will set K to {500 5}

#### Principles of the changes

* In tclParse.c :
  * The code from the patch of TIP282
  * In *ParseExpr* : The code to detect if a bareword is followed by a '=' sign and to add it to the literal list, so that it can be taken as a variable name during bytecode execution.
 
### Script shorthand

It can be found at [tcl-ExprShorthand-index-list-TIP282-script](https://github.com/florentis/tcl9.1-shorthand-index-list-TIP282-script). It includes all the previous additions, as well as the changes made for *scripted expression*. In a context where a script has to be compiled (proc, lambda), the shorthand should allow this syntax :

         proc P args {( *expression* )}

#### Principles of the changes
* In tclParse.c :
	* In *Tcl_ParseCommand* : At the begining, detect if the string begins with a `(` and finishes with a `)` then fills the parse structure with a TCL_TOKEN_SUB_EXPR, containing a TCL_TOKEN_SIMPLE_WORD.
 * In tclCompile.c :
 	* In *TclSetBytcodesFromAny* : Check if the string begins with a `(` and finishes with a `)`, then compile it as an *expression*. Avoid optimisation.
   	* In *TclCompileScript* :  Check if Tcl_ParseCommand returns a TCL_TOKEN_SUB_EXPR token. If so, compile its component as an expression


## Incompatibilities :
 * Main shorthand : Any proc which is named '(' will be shadowed by the shorthand. However, it will still be possible to use it, either by protecting it by a backslash '\\(', or simply, but less readable, by inserting a space between the open-bracket and the open-parenthese.
 
        set A [\( protect-it with backslash to eval the '(' proc ]
      
 * Index shorthand : As Tcl9 now forbid parentheses into array index, there should be none.
 
## Further developpements ideas :

- Allow barewords as variables ?

- **Implement prefixes** on operators and functions to allow richer infix expressions :

        protocol -regexp expr function {^(:{2}\w*)+$} {prefix funcName args} { 
           if {$prefix ne {}} {
             set func [namespace join $namespace $funcName]
             return [$func {*}$args]
           }
        }
        protocol -regexp expr operator {^(:{2}\w*)+$} {prefix opName left right} {
            if {$prefix ne {}} {
               set op [namespace join $namespace $opName]
               return [$op $left $right]
            }
        }

        set Modulus [( $z {::math::complexnumbers}* {::math::complexnumbers}conj($z) )]

- Implement the `{*}` prefix operator in expr to returns the prefixed word as a comma separated list. 

  	set L [list 1 2 3]
    set min [expr {min({*}$L)}]
  
## Copyright

This document has been placed in the public domain.









