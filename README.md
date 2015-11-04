How to get at R source. I am sick of Googling this. I am writing it down this time.

### TL;DR

``` r
methods(<S3_GENERIC>)
<S3_GENERIC>.default
<S3_GENERIC>.<CLASS>
getAnywhere(<S3_GENERIC>.<CLASS>)
getS3method("<S3_GENERIC>", "<CLASS>")
<NAMESPACE>:::<S3_GENERIC>.<CLASS>
```

### References

The definitive reference is this classic R News article:

Accessing the Sources
Uwe Ligges
<https://cran.r-project.org/doc/Rnews/Rnews_2006-4.pdf>
Volume 6/4, October 2006.
Go to page 43.

Another good reference is the help file for `method()`:

<https://stat.ethz.ch/R-manual/R-patched/library/utils/html/methods.html>

### Just print it

If you are lucky, just printing the function will work.

``` r
setNames
#> function (object = nm, nm) 
#> {
#>     names(object) <- nm
#>     object
#> }
#> <bytecode: 0x7fa8da3f0590>
#> <environment: namespace:stats>
```

But there are many ways this can fail.

``` r
vector             # .Internal
#> function (mode = "logical", length = 0L) 
#> .Internal(vector(mode, length))
#> <bytecode: 0x7fa8dbc63078>
#> <environment: namespace:base>
class              # .Primitive
#> function (x)  .Primitive("class")
subset             # S3 generic
#> function (x, ...) 
#> UseMethod("subset")
#> <bytecode: 0x7fa8dc2af790>
#> <environment: namespace:base>
```

What then?

### Function is an S3 generic

These are characterized by `UseMethod()` in the printed result:

``` r
subset
#> function (x, ...) 
#> UseMethod("subset")
#> <bytecode: 0x7fa8dc2af790>
#> <environment: namespace:base>
```

Print the default method by appending `.default`:

``` r
subset.default
#> function (x, subset, ...) 
#> {
#>     if (!is.logical(subset)) 
#>         stop("'subset' must be logical")
#>     x[subset & !is.na(subset)]
#> }
#> <bytecode: 0x7fa8dbf49190>
#> <environment: namespace:base>
```

Or list the methods for this generic:

``` r
methods(subset)
#> [1] subset.data.frame subset.default    subset.matrix    
#> see '?methods' for accessing help and source code
```

Then print the method you seek:

``` r
subset.matrix
#> function (x, subset, select, drop = FALSE, ...) 
#> {
#>     if (missing(select)) 
#>         vars <- TRUE
#>     else {
#>         nl <- as.list(1L:ncol(x))
#>         names(nl) <- colnames(x)
#>         vars <- eval(substitute(select), nl, parent.frame())
#>     }
#>     if (missing(subset)) 
#>         subset <- TRUE
#>     else if (!is.logical(subset)) 
#>         stop("'subset' must be logical")
#>     x[subset & !is.na(subset), vars, drop = drop]
#> }
#> <bytecode: 0x7fa8da303238>
#> <environment: namespace:base>
```

Sometimes the method definition is not exported from the package namespace. That's indicated by an asterisk `*` in the listing (pardon the way I do this, but I have to send through `capture.output()` if the asterisks are to survive `tail()`:

``` r
mout <- capture.output(methods(print))
tail(mout)
#> [1] "[188] print.vignette*                              "
#> [2] "[189] print.warnings                               "
#> [3] "[190] print.xgettext*                              "
#> [4] "[191] print.xngettext*                             "
#> [5] "[192] print.xtabs*                                 "
#> [6] "see '?methods' for accessing help and source code"
```

Good news: you've found the function you need. Bad news: you still can't read the source.

``` r
print.xgettext
#> Error in eval(expr, envir, enclos): object 'print.xgettext' not found
```

Use `getAnywhere()` or `getS3method()` to close the deal:

``` r
getAnywhere(print.xgettext)
#> A single object matching 'print.xgettext' was found
#> It was found in the following places
#>   registered S3 method for print from namespace tools
#>   namespace:tools
#> with value
#> 
#> function (x, ...) 
#> {
#>     cat(x, sep = "\n")
#>     invisible(x)
#> }
#> <bytecode: 0x7fa8dbb618b0>
#> <environment: namespace:tools>
```

This printed the source AND we learned the associated namespace. If you know the namespace, you can also use `:::` to see source:

``` r
tools:::print.xgettext
#> function (x, ...) 
#> {
#>     cat(x, sep = "\n")
#>     invisible(x)
#> }
#> <bytecode: 0x7fa8dbb618b0>
#> <environment: namespace:tools>
```

### Compiled code

Whenever you see `.C()`, `.Call()`, `.Fortran()`, `.External()`, or `.Internal()` and `.Primitive()`, the source you seek is in underlying compiled code.

You need to locate the source code of R or the associated add-on package on the internet or download it locally. Then browse around or search.

#### Visit the source of R on the internet:

-   A read-only mirror of the R source on GitHub: <https://github.com/wch/r-source>
-   A rudimentary web interface is available from the SVN repository: <https://svn.r-project.org/R/>.

#### Download source of R itself:

-   Download source for current release from <https://cran.r-project.org>, e.g. `R-3.2.2.tar.gz`, and unpack it.
     tar xvf R-3.2.2.tar.gz
-   See more info there about getting the development version.
-   Or checkout from the official Subversion repository <https://svn.r-project.org/R/>.

#### How to find what you need in the R source (paraphrasing Ligges, in places):

-   For R and standard R packages, look in subdirs of `$R_HOME/src/`, most especially `$R_HOME/src/main/`.
-   If calling R function is `.Primitive()` or `.Internal()`, find the entry point `$R HOME/src/main/names.c`. Then try to find that function. Example below.
-   Use the [GitHub search capabilities](https://help.github.com/articles/searching-github/).

Example: I want the source for `levels<-`.

``` r
`levels<-`    # .Primitive()
#> function (x, value)  .Primitive("levels<-")
```

Search for `levels<-` in `$R HOME/src/main/names.c` and we find [this line](`$R%20HOME/src/main/names.c`):

``` c
{"levels<-", do_levelsgets, 0, 1, 2, {PP_FUNCALL, PREC_LEFT, 1}}
```

which tells we're looking for `do_levelsgets`. Now use your search capability (GitHub? `grep`?) to look for that within the files below `$R_HOME/src/`. I choose the [GitHub option](https://github.com/wch/r-source/search?utf8=✓&q=+do_levelsgets+path%3Asrc%2Fmain&type=Code) and use this query:

``` bash
 do_levelsgets path:src/main
```

And finally arrive at my destination: [lines 1242 through 1261 in `$R_HOME/src/main/attrib.c`](https://github.com/wch/r-source/blob/f42ee5e7ecf89a245afd6619b46483f1e3594ab7/src/main/attrib.c#L1242-L1261).

#### Visit the source of an R package on the internet:

-   If developed on GitHub, go to package's official repo. Ideally, this will be provided as a URL on the package's CRAN page, but sadly not always the case.
    -   Example: <https://github.com/vegandevs/vegan>
-   If the package is on CRAN but not on GitHub, go to the read-only mirror of its source from the [METACRAN](https://github.com/cran) project.
    -   Example: <https://github.com/cran/MASS>

#### Download source of an add-on package:

-   Visit the package on CRAN, e.g., <https://cran.r-project.org/web/packages/vegan/index.html>, download the `*.tar.gz` linked from "Package source", and unpack it.
     tar xvf vegan\_2.3-1.tar.gz
-   If developed on GitHub, e.g., <https://github.com/vegandevs/vegan>, download via the "Download ZIP" button or Git clone it.
     git clone <https://github.com/vegandevs/vegan.git>

#### How to find what you need in R package source:

-   Look in the `src` directory. If you are lucky, there will be a file that obviously contains the function of interest.
-   Otherwise, search with `grep`, your editor/IDE, or [GitHub queries](https://help.github.com/articles/searching-github/) and follow the trail to rainbow's end.

Example: I want the source for `dplyr::bind_rows`.

First I search within the `R` directory with [this GitHub search query](https://github.com/hadley/dplyr/search?utf8=✓&q=bind_rows+path%3AR):

``` bash
bind_rows path:R
```

GitHub search only shows you the first one or two matches within a file, but I gather that `R/bind.r` is where I want to look. I [visit it in the browser](https://github.com/hadley/dplyr/blob/master/R/bind.r) and use my browser to search for `bind_rows`, which reveals the [function definition](https://github.com/hadley/dplyr/blob/master/R/bind.r). That reveals I actually need `bind_rows_`.

So I do another [GitHub search with this query](https://github.com/hadley/dplyr/search?utf8=✓&q=bind_rows_):

``` bash
bind_rows_
```

which reveals hits in

    R/RcppExports.R
    src/bind.cpp
    src/RcppExports.cpp

Reading the [relevant bit of `src/bind.cpp`](https://github.com/hadley/dplyr/blob/2a3d526a85f484f75218807e0e5feed2d65e2c13/src/bind.cpp#L122-L124) reveals I really need `rbind__impl`.

So I do another [GitHub search with this query](https://github.com/hadley/dplyr/search?utf8=✓&q=rbind__impl):

``` bash
rbind__impl
```

And finally arrive at my destination: [lines 7 though 119 of `src/bind.cpp`](https://github.com/hadley/dplyr/blob/2a3d526a85f484f75218807e0e5feed2d65e2c13/src/bind.cpp#L7-L119).

#### Things I haven't covered

An incomplete list:

-   S4
-   [R-Forge](https://r-forge.r-project.org)
