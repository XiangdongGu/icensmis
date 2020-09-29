## Rcpp randmo number Issue for newer versions

The cpp code
```
#include <Rcpp.h>
using namespace Rcpp;

// [[Rcpp::export]]
double loglik(NumericVector y, NumericMatrix X, NumericVector beta) {
  int J = X.ncol(), n = X.nrow();
  double result = 0.0, temp;
  for (int i=0; i < n; i++) {
    temp = 0;
    for (int j=0; j < J; j++) {
      temp += X(i, j) * beta[j];
    }
    result += (y[i] - temp)*(y[i] - temp);
  }
  return result;  
}

// [[Rcpp::export]]
NumericVector gradlik(NumericVector y, NumericMatrix X, NumericVector beta) {
  int J = X.ncol(), n = X.nrow(), i, j;
  double temp;
  NumericVector result(J);
  for (i=0; i<n; i++) {
    temp = 0;
    for (j=0; j<J; j++) {
      temp += X(i, j) * beta[j];
    }
    temp += - y[i];
    for (j=0; j<J; j++) {
      result[j] += 2* X(i, j) * temp;
    }
  }
  return result;
}

// [[Rcpp::export]]
void test(NumericMatrix X, NumericVector y, Function fitfun) {
  RNGScope scope;
  int J = X.ncol(), i;
  NumericVector par(J);
  for (i=0; i<J; i++) par[i] = 0;
  for (int i=0; i<10; i++) {
    Rprintf("%8f\n", log(R::runif(0.0, 1.0)));
    par = fitfun(par, y, X);
  }
}
```

The R code

```
source("rcpp_new/.Rprofile", chdir = TRUE)
#source("rcpp_old/.Rprofile", chdir = TRUE)
library(Rcpp)
sourceCpp("test.cpp")

# Generate Test Data-----------------------------------------------------------
set.seed(123)
x <- rnorm(10)
y <- x + rnorm(10, sd = 0.1)
X <- cbind(1, x)

# Optimization function--------------------------------------------------------
fitfun <- function(parm, y, X) {
  q <- optim(parm, loglik, gradlik, method="BFGS", X = X, y = y)
  q$par
}

# New version Rcpp: the random number never change
# Old version Rcpp: the random number will change
test(X, y, fitfun)
```

The issue here is that the random number generated is not changing if we pass an R function calling Rcpp function and into Rcpp. This is rare use case, but unfortunately used in our package.

See the following [Github issue](https://github.com/RcppCore/Rcpp/issues/871) for discussion.

## Possible solution
add GetRNGstate() and PutRNGstate() for each call of random function.