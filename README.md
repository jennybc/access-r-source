Accessing R Source
================

-   [TL;DR](#tldr)
-   [References](#references)
-   [Just print it](#just-print-it)
-   [Function is an S3 generic](#function-is-an-s3-generic)
-   [Compiled code](#compiled-code)

*2017-07-31 update: Since I wrote this @jimhester has created the [lookup package](https://github.com/jimhester/lookup#readme) to automate this process. So if all you want is the result, just use that! If you want a bit more context, then keep reading. AFAIK this info is still fundamentally sound.*

How to get at R source. I am sick of Googling this. I am writing it down this time.

### TL;DR

``` r
methods(<S3_GENERIC>)
<S3_GENERIC>.default
<S3_GENERIC>.<CLASS>
getAnywhere(<S3_GENERIC>.<CLASS>)
getS3method("<S3_GENERIC>", "<CLASS>")
<NAMESPACE>:::<S3_GENERIC>.<CLASS>
pryr::show_c_source(.Primitive("class")) # compiled code
```

### References

The definitive reference is this classic R News article:

> Accessing the Sources
>
> Uwe Ligges
>
> <https://cran.r-project.org/doc/Rnews/Rnews_2006-4.pdf>
>
> Volume 6/4, October 2006. Go to page 43.

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
#> <bytecode: 0x7fce93c98920>
#> <environment: namespace:stats>
```

But there are many ways this can fail.

``` r
vector             # .Internal
#> function (mode = "logical", length = 0L) 
#> .Internal(vector(mode, length))
#> <bytecode: 0x7fce94835c40>
#> <environment: namespace:base>
class              # .Primitive
#> function (x)  .Primitive("class")
subset             # S3 generic
#> function (x, ...) 
#> UseMethod("subset")
#> <bytecode: 0x7fce933c0178>
#> <environment: namespace:base>
```

What then?

### Function is an S3 generic

These are characterized by `UseMethod()` in the printed result:

``` r
subset
#> function (x, ...) 
#> UseMethod("subset")
#> <bytecode: 0x7fce933c0178>
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
#> <bytecode: 0x7fce935f1228>
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
#> <bytecode: 0x7fce93314388>
#> <environment: namespace:base>
```

Sometimes the method definition is not exported from the package namespace. That's indicated by an asterisk `*` in the listing (pardon the way I do this, but I have to send through `capture.output()` if the asterisks are to survive `tail()`:

``` r
mout <- capture.output(methods(print))
tail(mout)
#> [1] "[195] print.vignette*                                   "
#> [2] "[196] print.warnings                                    "
#> [3] "[197] print.xgettext*                                   "
#> [4] "[198] print.xngettext*                                  "
#> [5] "[199] print.xtabs*                                      "
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
#> <bytecode: 0x7fce9787e150>
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
#> <bytecode: 0x7fce9787e150>
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

For quick results, try `pryr::show_c_source()` from the [`pryr`](https://cran.r-project.org/web/packages/pryr/index.html) package. It will open your default web browser and search the GitHub R source repo for the function's code.

Example: I want the source for `levels<-`.

``` r
`levels<-`    # .Primitive()
#> function (x, value)  .Primitive("levels<-")
```

Copy and paste the function body to `show_c_source()`

``` r
pryr::show_c_source(.Primitive("levels<-"))
#> levels<- is implemented by do_levelsgets with op = 0
#> Please visit https://github.com/search?q=SEXP%20attribute_hidden%20do_levelsgets+repo:wch/r-source&type=Code
```

Alternatively, by hand:

-   For R and standard R packages, look in subdirs of `$R_HOME/src/`, most especially `$R_HOME/src/main/`.
-   If calling R function is `.Primitive()` or `.Internal()`, find the entry point `$R HOME/src/main/names.c`. Then try to find that function. Example below.
-   Use the [GitHub search capabilities](https://help.github.com/articles/searching-github/).

Example (continued):

Search for `levels<-` in [`$R HOME/src/main/names.c`](https://github.com/wch/r-source/blob/trunk/src/main/names.c) and we find [this line](https://github.com/wch/r-source/blob/3d8fc77d602f19c9722d8e13c1a1f5f69f42a5c4/src/main/names.c#L218):

``` c
{"levels<-", do_levelsgets, 0, 1, 2, {PP_FUNCALL, PREC_LEFT, 1}}
```

which tells us we're looking for `do_levelsgets`. Now use your search capability (GitHub? `grep`?) to look for that within the files below `$R_HOME/src/`. I choose the [GitHub option](https://github.com/wch/r-source/search?utf8=✓&q=+do_levelsgets+path%3Asrc%2Fmain&type=Code) and use this query:

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

        tar xvf vegan_2.3-1.tar.gz

-   If developed on GitHub, e.g., <https://github.com/vegandevs/vegan>, download via the "Download ZIP" button or Git clone it.

        git clone https://github.com/vegandevs/vegan.git

#### How to find what you need in R package source:

-   Look in the `src` directory. If you are lucky, there will be a file that obviously contains the function of interest.
-   Otherwise, search with `grep`, your editor/IDE, or [GitHub queries](https://help.github.com/articles/searching-github/) and follow the trail to rainbow's end.

Example: I want the source for `dplyr::bind_rows`. `dplyr` is developed [on GitHub](https://github.com/hadley/dplyr).

First I search within the `R` directory with [this GitHub search query](https://github.com/hadley/dplyr/search?utf8=✓&q=bind_rows+path%3AR):

``` bash
bind_rows path:R
```

GitHub search only shows you the first one or two matches within a file, but I gather that [`R/bind.r`](https://github.com/hadley/dplyr/blob/master/R/bind.r) is where I want to look. I visit it in the browser and use the browser to search for `bind_rows`, which reveals the [function definition](https://github.com/hadley/dplyr/blob/969a3a8e2f1dea5802a1cb6c13e189691a0e206f/R/bind.r#L70-L81). That reveals I actually need `bind_rows_`.

So I do another [GitHub search with this query](https://github.com/hadley/dplyr/search?utf8=✓&q=bind_rows_):

``` bash
bind_rows_
```

which reveals hits in

    R/bind.r
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
-   Other places to put code, e.g. [R-Forge](https://r-forge.r-project.org) or BitBucket
