
# Rootfinding

In this notebook I will do the speed comparisons.

We want to find the roots of the following real functions $f_i : \mathbb{R} \to \mathbb{R}$.

- $f_1(x) = f_1(x) = 0.5 x^{-0.5} + 0.5 x^{-0.2}$
- $f_2(x) = \log(a+x) - b$ for $(a, b) = 1.5, 1$
- $f_3(x) = - U'(a_0 + y_0 - x) + \beta (1+r) U'((1+r)\cdot x + y_1) $
for $U'(c) = c^{-\sigma}$ and $(a_0, y_0, y_1, r, \beta, \sigma) = (1, 1, 1, 0.03, 0.99, 2)$.

- $ f_4(x) = - U'(y_0 + a_0 - x) + \beta (1+r) \frac{1}{2} \bigl( U'(y_L + (1+r) x) + U'(y_H + (1+r) x))\bigr) $
where $U'(c) = c^{-\sigma}$, $(a_0, y_0, \bar y, r, \beta, \sigma) = (1, 1, 1, 0.03, 0.99, 2)$ and $\lambda \in \{ 0, 0.25, 0.5, 0.75 \}$.

These functions are found in `testfunctions.jl` and in the module `testfunctions.f90`.

We are using the methods 
1. bisection
2. newton
3. brent (Fortran Code by [Tony Smith](www.econ.yale.edu/smith/)).

The corresponding functions can be found in the files `rootfinding.jl` and `rootfinding.f90`.

## Description of the program
```
for i = 1:1000
    f_2 > bisect, newton, brent
    f_3 > bisect, brent
    f_4(λ) > bisect, brent λ = 0.25
end
```




```julia
include("Julia/testfunctions.jl")
include("Julia/rootfinding.jl")
```




    zbrent (generic function with 1 method)



The `Julia` version of the test program is provided below.


```julia
function test(dobrent::Bool; verbose=false)
    res2 = zeros(3)
    res3 = zeros(2)
    for i = 1:1000 
        x = 2.0
        xlow2 = 0.001
        xhigh2 = 5
        iterations = 40
        toler = 1e-10

        res2[1] = bisect(f2, xlow2, xhigh2, mxiter=iterations, toler=toler,verbose=verbose)
        res2[2] = newton(f2, f2p, xhigh2, mxiter=iterations, toler=toler, verbose=verbose)
        if dobrent
            res2[3] = zbrent(f2, xlow2, xhigh2, rtol=toler, ftol=toler, itmax=iterations, verbose=verbose)
        end
        
        xlow3, xhigh3 = 1e-5, 1.99
        res3[1] = bisect(f3, xlow3, xhigh3, mxiter=iterations, toler=toler, verbose=verbose)
        if dobrent
            res3[2] = zbrent(f3, xlow3, xhigh3, rtol=toler, ftol=toler, itmax=iterations, verbose=verbose)
        end
        #λ_vec = 0:0.25:0.75
        res4 = zeros(2)

        #for (i, λ) in enumerate(λ_vec)
#        if do4
            res4[1] = bisect(x::Real -> f4(x, 0.25), xlow3, xhigh3, mxiter=iterations, toler=toler, verbose=verbose)
            if dobrent 
                res4[2] = zbrent(x::Real -> f4(x, 0.25), xlow3, xhigh3, rtol=toler, ftol=toler, itmax=iterations, verbose=verbose)
            end
#        end
        #end
        if verbose
            mean2 = mean(res2)
            mean3 = mean(res3)
            mean4 = mean(res4)
            print("$mean2, $mean3, $mean4, $xlow2, $xhigh2\n")
        end
    end
end

# warm up
test(true)
```

I also provide the main program of the `Fortran` version (see also `test.f90`) for convenience.

```fortran
program main
  
  use constants
  use functions
  use rootfinding
 
  real(ndp) :: x = 2.0
  real(ndp) :: xlow2, xhigh2, xlow3, xhigh3, toler
  integer(i4b) :: i, iterations
  logical :: verbose = .false.
  
  real(ndp) :: res2(3), res3(2), res4(2)

  do i = 1,1000
     xlow2 = 0.001
     xhigh2 = 5
     iterations = 60
     toler = 1d-10

     res2(1) = bisect(f2, xlow2, xhigh2, iterations, toler, verbose)
     res2(2) = newton(f2, f2p, xhigh2, iterations, toler, verbose)
     res2(3) = zbrent(f2, xlow2, xhigh2, toler, toler, iterations, verbose)
     if (verbose) then
        write(6,"('f2: 'f15.8)") sum(res2)/3
     endif

     xlow3 = 0.1
     xhigh3 = 1.99
     res3(1) = bisect(f3, xlow3, xhigh3, iterations, toler, verbose)
     res3(2) = zbrent(f3, xlow3, xhigh3, toler, toler, iterations, verbose)
     if (verbose) then
        write(6,"('f3: 'f15.8)") sum(res3)/2
     endif

     res4(1) = bisect(f4, xlow3, xhigh3, iterations, toler, verbose)
     res4(2) = zbrent(f4, xlow3, xhigh3, toler, toler, iterations, verbose)
     if (verbose) then
        write(6,"('f4: 'f15.8)") sum(res4)/2
     endif
  enddo

  end program main
```

Call the `makefile` to compile the code.


```julia
; cd Fortran90
```

    /Users/Fabian/Dropbox/Yale_Courses/Comp_Econ/Julia-for-Computational-Economics/Rootfinding/Fortran90



```julia
; make
```

    gfortran -c constants.f90
    gfortran -c testfunctions.f90
    gfortran -c rootfinding.f90
    gfortran -c test.f90
    gfortran -o test.out test.o rootfinding.o testfunctions.o constants.o



```julia
; cd ..
```

    /Users/Fabian/Dropbox/Yale_Courses/Comp_Econ/Julia-for-Computational-Economics/Rootfinding



```julia
N = 100
time_julia = zeros(N)
time_fortran = zeros(N)

for i = 1:N
    time_fortran[i] = @elapsed run(`./Fortran90/test.out`)
    time_julia[i] = @elapsed test(true)
end
```


```julia
using Plots, PyPlot
```

    /Users/Fabian/.julia/Conda/deps/usr/lib/python2.7/site-packages/matplotlib/font_manager.py:273: UserWarning: Matplotlib is building the font cache using fc-list. This may take a moment.
      warnings.warn('Matplotlib is building the font cache using fc-list. This may take a moment.')



```julia
pyplot()

histogram(time_fortran, label="Fortran", alpha=0.5)
histogram!(time_julia, label="Julia", alpha=0.5)

plot!([median(time_fortran);median(time_julia)], linetype=:vline, label="medians")
#hline!(median(time_fortran))
#hline!(median(time_julia))
```




<img src="data:image/png;base64,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AADAZhQsAAAAm1GwAAdavny56QgALGBWwxcFC3CgtLQ00xEAWMCshi8KFuBAS5cuNR0BgAXMaviiYAEAANiMggUAAGAzChbgQOXl5aYjALCAWQ1fvRas1tZWfetb39JFF12kzMxMzZkzRxUVFZKkuro6XX/99Zo0aZKmTZumd955Z0ADA5Dy8/NNRwBgAbMavs74DtaSJUtUXl6u0tJS5eTk6Hvf+54k6aGHHtLMmTN14MABPffcc7rzzjvV0dExYIEBSOvXrzcdAYAFzGr46rVgRUdH67rrruu6PmPGDB0+fFiS9OKLL+qee+6RJF1++eUaP3683n777QGICuAkvvoNOAOzGr4s7YO1bt063XLLLTp27Jja29s1duzYrr+bMGGCqqqqQhYQAADAaYb1tcDq1atVUVGhX/7yl/L5fAORCQAAwNHO+g7W448/rqKiIr3++uuKiYnRqFGjNGzYMNXW1nYtU1lZ2edboHPnzlVOTk63y8yZM1VUVNRtue3bt2vhwoU91n/11VdVWlra7baj1dXavLmwR+l77LHHtGbNmm63VVVVKScnp8e3OZ566qkepzHw+XzKyclRcXFxt9sLCwu1aNGiHtnmz5/f63bk5OT0WPZHP/qRNm7c2OP2ZcuWye12d7tt5cqVjtqOkpIS5eTksB0DtB2nZjGxHaWlH9iyHady8s+D7WA7zrQda9asGRLbIQ2Nn4cd21FYWKj09HRNnz69q9Pcf//9Pe4vIhAIBHrcKumJJ57QCy+8oDfffFNJSUldty9evFgTJkzQypUrtWvXLt16662qrKxUVFRUj/soKSnRZZddpt27dysrK6u3h+mhurpaG1av0oIZF2vcqGRL60jS0WMN+s27H+pHP16p1NRUy+uZdOBYhb7/2j/r2euf0KRRGabjwEFWrlypVatWGXt8XruANaZnFQOjt77T60eEn332mR588EFlZGTo6quvViAQUExMjP73f/9Xjz76qBYsWKBJkyYpOjpav/3tb3stVwBCh1/YgDMwq+Gr14I1fvx4dXZ29rrC2LFj9cYbb4Q0FAAAgJNxJHcAAACbUbAABzp9Z1AAgxOzGr4oWIADLV682HQEABYwq+GLggU4UEFBgekIACxgVsMXBQtwIKuHPQFgFrMavihYAAAANqNgAQAA2IyCBThQb6dcAjD4MKvhi4IFOFBJSYnp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TwrmNRPs8qG87/48N6011dq7d6+qq6vPuNzJnnOy90iSAkGqra0NJCUlBTo6OrpuGzduXKCioqLHsps2bQpI4sKFCxcuXLhwGfKXTZs2dXWgoN/BGjNmjLKysvSb3/xGCxcu1EsvvaQLLrhAF154YY9l58yZo02bNmnixImKjT3z/1wBAACcqrm5WZWVlZozZ07XbRGBs71fewYHDhzQd7/7XX3++edKSkrSr371K1188cW2hgUAAHCqfhUsAAAAnBlHcgcAALAZBQswxOoZEX7/+99rypQpmjx5subNmyev19v1d5GRkbr00kuVmZmprKws7TzltByccQGwx7nO6v79+7tmNCsrS+np6UpJ+fu3QSdOnKgpU6Z0LfPiiy8OyHYhxIL9FiEAe1xzzTWBX//614FAIBB46aWXAldccUWPZbxeb+ALX/hC4MCBA4FAIBDIy8sLLF++vOvvIyMjA8ePH+/3/QPomx2zeqq8vLzAvffe23U9PT09sHfv3hAkh0kULMAAq4c7efHFFwPXX3991/WPPvoocP7553ddj4iICDQ2Nvb7/gGcnV2zelJLS0tg5MiRgT179nTdNnHixG7XMTTwESFggNUzIlRVVWnChAld1ydOnKjq6uqug+pFRETo6quvVmZmph588EH5fL6g7h/A2Z3LrB49erTHATBffvllZWRkaNq0ad1uX7BggS699FJ9//vfl9vtDsGWYKBRsACHiYiI6Prz4cOHtXv3bv35z39WbW2t8vPzDSYD0JfnnntOd999d7fb3nnnHe3Zs0clJSUaPXq0Fi5caCgd7ETBAgy44IILur0TJZ34H3BaWlq35dLS0lRZWdl1/dChQ93+N33++edLkmJjY/VP//RPeuedd4K6fwBnZ9esSlJlZaXee+893Xnnnd3WPTnHUVFRuv/++1VcXByCLcFAo2ABBpx6RgRJZzwjwnXXXafS0lIdOHBAkvT000/r29/+tiSpoaGh67xXnZ2d2rJlizIzM4O6fwBnZ8esnrRx40Z961vfUmLi38+z6PP51NjY2HX9hRde6JpjOBsHGgUMOf2MCP/xH/+hL3/5y1q5cqXGjx+vH/zgB5JOfPV7+fLl6ujo0CWXXKLnn39eCQkJevfdd7VkyRJFRkaqvb1dWVlZWrdunZKTk3u9f864APTPuc6qJAUCAU2cOFG/+c1vdOWVV3bd96FDh3Tbbbeps7NTgUBAF154odatW8e7zUMABQsAAMBmfEQIAABgMwoWAACAzShYAAAANqNgAQAA2IyCBQAAYLP/Dwee6JOdD0fyAAAAAElFTkSuQmCC" />




```julia

```