---
title: "Gradient descent for the elastic net Cox-PH model"
output: html_document
fontsize: 12pt
published: true
status: publish
mathjax: true
---
 
$$
\newcommand{\bbeta}{\boldsymbol{\beta}}
\newcommand{\bx}{\boldsymbol{x}}
\newcommand{\bX}{\boldsymbol{X}}
\newcommand{\bY}{\boldsymbol{Y}}
\newcommand{\bW}{\boldsymbol{W}}
\newcommand{\bp}{\boldsymbol{p}}
\newcommand{\etab}{\boldsymbol{\eta}}
\newcommand{\bsigma}{\boldsymbol{\sigma}}
\newcommand{\bP}{\boldsymbol{P}}
\newcommand{\bdelta}{\boldsymbol{\delta}}
\newcommand{\bw}{\boldsymbol{w}}
\newcommand{\bxi}{\bx_i}
\newcommand{\ei}{\varepsilon_i}
$$
 
## Introduction
 
Machine learning in the survival modelling context is relatively small area of research, but has been gaining attention in recent years. See this [arXiv paper](https://arxiv.org/abs/1708.04649) for a good overview. Unlike the classical supervised learning scenario where a dataset contains $N$ labelled observations of the form $D=\\{(y_i,\bxi) \\}_{i=1}^{N}$, where $y_i$ is the labelled response and $\bxi$ a $p$-dimensional feature vector, a survival model contains $N$ partially labelled observations of the following tuple: $(t_i,\delta_i,\bxi)$, where $t_i$ is the last recorded time point, and $\delta_i$ is a censoring indicator equal to 1 if the event of interest happened at that time point and 0 otherwise. When $\delta_i=0$ the observation is said to be censored and that person $i$ will experience the event at some point $\geq t_i$. 
 
To someone unfamiliar with survival models, it may be unclear as to what the regression/classification task is. Should the goal be to learn a hypothesis such that $h(x_i) \approx t_i$? This is problematic because if a patient is censored, how do you measure the loss of a prediction $\hat{t}_i$? Clearly it is bad to predict $\hat{t}_i < t_i$, but how can we penalize $\hat{t}_i > t_i \| \delta_i=0$ since we don't know what the final event time $t_i$ will be?
 
One technique is convert the survival response data into a binary classification problem. For example we could predict whether a patient will be alive at some time point $\bar{t}$. Picking the right time point can be tricky though. When $\bar{t}$ is low, then most patients will be alive at that point and so the binary response vector will be highly imbalanced. In contrast, for later points, patients which were censored before $\bar{t}$ will be excluded. In other words as $\bar{t}$ increases, the size of the dataset will shrink (if there is censoring). Often times an average of several time points is calculated, such as the time-dependent AUC scores, allowing for a more robust assessment.
 
The most common evaluation metrics to measure generalization accuracy in survival modelling are discriminative assessments, such as the concordance index (c-index) score, to evaluate how a predicted estimates discriminates *between* patients. In terms of a survival process it is useful to think in terms of a hazard score $h_i$, where $h_i > h_j$ implies that patient $i$ is more likely to experience the event before patient $j$. More concretely the concordance probability between patient $i$ and $j$ can be written: $c_{ij} = P(h_i > h_j \| t_i \geq t_j)$, and the concordance index can be calculated by summing over all possible comparisons and using an indicator to measure if the pairwise comparisons are concordant (i.e. $i$ dies before $j$ and $i$ has a higher risk score than $j$).
 
$$
\begin{align*}
\text{c-index} &= \frac{1}{M} \sum_{i:\delta_i=1} \sum_{j:t_i < t_j} I\{ h_i > h_j \}
\end{align*}
$$
 
It should be noted that the magnitude of the difference between the time points or the hazard scores are irrelevant for the c-index. Hence a good function will provide risk scores that "discriminate" between patients in terms of who dies first.
 
## Cox partial likelihood
 
Unfortunately the c-index is a non-convex loss function and hence its optimization is NP-hard. Just as the logistic loss function is used as a convex approximation of overall accuracy, the partial likelihood from the Cox-PH model can be used as a convex approximation of the c-index. In the notation that follows we will replace a patient's risk score with a linear combination of features: $h(\bxi)=\bxi^T\bbeta$, and $y_j(t_i)$ is an indicator if patient $j$ is alive at time $t_i$.
 
$$
\begin{align}
&\text{Partial Likelihood} \nonumber \\
L(\bbeta) &= \prod_{i=1}^N \Bigg( \frac{e^{\bxi^T \bbeta}}{\sum_{j=1}^N y_j(t_i) e^{\bx_j^T \bbeta}} \Bigg)^{\delta_i} \label{eq:partial} \\
&\text{Partial Log-Likelihood} \nonumber \\
\ell(\bbeta) &= \sum_{i=1}^N \delta_i \Bigg\{ \bxi^T \bbeta - \log \Bigg[\sum_{j=1}^N y_j(t_i) \exp(\bx_j^T \bbeta) \Bigg] \Bigg\} \label{eq:logpartial}
\end{align}
$$
 
Several things are worth noting about the partial likelihood function in equation \eqref{eq:partial}. First, it is very similar to the [softmax function](https://en.wikipedia.org/wiki/Softmax_function), with the only difference being that sum of terms in the denominator is changing across the product terms, meaning the sums of these terms does not need to add up to one. Additionally, it is the product of only $\sum_i \delta_i$ terms, since if $\delta_i=0$, it is equal to one. However, censored patients still impact the calculation because their risk scores can appear in the denominator. 
 
We can use the partial log-likelihood shown in equation \eqref{eq:logpartial} to compute the derivative with respect to the parameters in $\bbeta$. However, it is easier to first solve the partial likelihood with respect to the risk scores $\eta_i=\bxi^T\bbeta$ and then obtain the complete derivative. 
 
$$
\begin{align*}
\ell(\eta) &= \sum_{i=1}^N \delta_i \Bigg\{ \eta_i - \log \Bigg[\sum_{j=1}^N y_j(t_i) \exp(\eta_j) \Bigg] \Bigg\} \\
\frac{\partial \ell(\eta)}{\partial \eta_q} &= \delta_q - \sum_{i=1}^N\delta_i \Bigg(\frac{y_q(t_i)\exp(\eta_q)}{\sum_{j=1}^N y_j(t_i) \exp(\eta_j)} \Bigg) \\
&= \delta_q - \sum_{i=1}^N\delta_i \pi_{qi}
\end{align*}
$$
 
Where $\pi_{ij}$ represents person $i$'s relative risk score at time $j$, or alternatively their softmax score at time $t_j$. If the observations are ordered so that $t_1 < \dots < t_N$ then $\bP$ will be a lower triangular matrix where $\bP_{ij}=\pi_{ij}$. Finally, to recover $\frac{\partial \ell}{\partial \bbeta}$ notice that:
 
$$
\begin{align}
\frac{\partial \eta}{\partial \bbeta^T} &= \begin{bmatrix} \frac{\partial \eta_1}{\partial \beta_1} &  \cdots  & \frac{\partial \eta_N}{\partial \beta_1} \nonumber \\
\vdots & \cdots & \vdots \nonumber \\ 
\frac{\partial \eta_1}{\partial \beta_p} & \cdots & \frac{\partial \eta_N}{\partial \beta_p} \end{bmatrix}  = \begin{bmatrix} x_{11} &  \cdots  & x_{N1} \nonumber \\
\vdots & \cdots & \vdots \nonumber \\ 
x_{1p} & \cdots & x_{Np} \end{bmatrix} = \bX^T \nonumber \\
\frac{\partial \ell(\bbeta)}{\partial \bbeta} &= \frac{\partial \eta}{\partial \bbeta^T} \frac{\ell(\bbeta)}{\partial \eta} \nonumber \\
&= \bX^T (\bdelta - \bP \bdelta) \label{eq:coxgrad}
\end{align}
$$
 
Hence gradient descent can performed by using equation \eqref{eq:coxgrad}. Unfortunately compared to other convex loss functions, the partial likelihood gradient is slower to update because $\bP \bdelta$ requires $O(N^2)$ calculations, which is inevitable due to the double sum in the loss function. One will need to update the lower-triangular matrix $\bP$ every iteration. 
 
## Gradient descent with regularization 
 
An elastic net regularization term is easily added in this set up so that gradient of the penalized loss function becomes:
 
$$
\begin{align*}
\mathcal{p}\ell(\bbeta ; \lambda) &= -\ell(\bbeta) + P(\lambda,\bbeta) \nonumber \\
\frac{\mathcal{p}\ell(\bbeta ; \lambda)}{\partial \bbeta} &= -\bX^T(\bdelta - \bP\bdelta) + \partial_{\bbeta} P(\lambda,\bbeta)
\end{align*}
$$
 
For example, if $P(\lambda,\bbeta)= \lambda 0.5 \\| \bbeta \\|_{2}^2$, then the gradient descent update will become:
 
$$
\begin{align*}
&\text{Ridge-Cox GD update} \\
\bbeta^{(k)} &=\beta^{(k-1)} - \gamma \frac{\mathcal{p}\ell(\bbeta^{(k-1)} ; \lambda)}{\partial \bbeta}  \\
\bbeta^{(k)} &=\beta^{(k-1)} + \frac{\gamma}{N}\bX^T(\bdelta - \bP\bdelta) - \gamma\lambda \bbeta^{(k-1)} \\
\end{align*}
$$
 
We can check that this will achieve the same results as `glmnet`:
 

{% highlight r %}
library(glmnet)
library(survival)
df <- survival::veteran
df <- df[order(df$time),]
df <- df[!duplicated(df$time),]
delta <- df$status
time <- df$time
So <- Surv(time=time,event=delta)
X <- as.matrix(df[,c('karno','diagtime','age')])
Xscale <- scale(X)
N <- nrow(Xscale)
 
# glmnet
lam <- 1
alpha <- 0
mdl.glmnet <- glmnet(x=Xscale,y=So,family='cox',alpha=alpha,lambda=lam,standardize = F)
 
# Gradient descent
gam <- 0.1
beta.ridge <- as.matrix(rep(0,ncol(Xscale)))
for (k in 1:250) {
  eta <- as.vector(Xscale %*% beta.ridge)
  haz <- as.numeric(exp(eta))
  rsk <- rev(cumsum(rev(haz)))
  P <- outer(haz, rsk, '/')
  P[upper.tri(P)] <- 0
  beta.ridge <- beta.ridge + (gam/N)*(t(Xscale) %*% (delta - P %*% delta)) - (gam*lam*beta.ridge)
}
 
round(data.frame(glmnet=coef(mdl.glmnet)[,1],beta.ridge),4)
{% endhighlight %}



{% highlight text %}
##           glmnet beta.ridge
## karno    -0.2288    -0.2288
## diagtime  0.0577     0.0577
## age       0.0173     0.0173
{% endhighlight %}
 
Our results will differ slightly from `glmnet` when ties are not properly taken account of in the construction of our $\bP$ matrix (see [here](https://en.wikipedia.org/wiki/Proportional_hazards_model#Tied_times) for a discussion of tied times). For this reason ties have been removed from the dataset, but they are not challenging to incorporate. When $P(\lambda,\bbeta) = \lambda \\| \bbeta \\|_1$, which is the Lasso penalty term, we will have to use proximal gradient descent due to non-smooth (but convex) penalty term. For a discussion of how to obtain the solution to the proximal mapping in the Lasso case see [here](http://www.stat.cmu.edu/~ryantibs/convexopt-S15/scribes/08-prox-grad-scribed.pdf).
 
$$
\begin{align*}
&\text{Lasso-Cox Proximal-GD update} \\
\bbeta^{(k)} &= S_{\gamma\lambda}\Bigg(\beta^{(k-1)} + \gamma \frac{\ell(\bbeta)}{\partial \bbeta}\Bigg) \\ 
&= S_{\gamma\lambda}\Bigg(\beta^{(k-1)} + \frac{\gamma}{N}\bX^T(\bdelta - \bP\bdelta)\Bigg) \\ 
S_{r}(x) &= \begin{cases}
x - r & \text{ if } x > r \\
0 & \text{ if } |x| \leq r \\
x + r & \text{ if } x < -r \\
\end{cases}
\\
\end{align*}
$$
 
Where $S()$ is the soft-thresholding function. Again we can check that this proximal updating scheme will replicate `glmnet`.
 

{% highlight r %}
# glmnet
lam <- 1/70
alpha <- 1
mdl.glmnet <- glmnet(x=Xscale,y=So,family='cox',alpha=alpha,lambda=lam,standardize = F)
 
# proximal gradient descent
Softfun <- function(x,r) {
  ifelse(x > r,x - r,ifelse(-x > r,x + r,0))
}
 
gam <- 0.1
beta.lasso <- as.matrix(rep(0,ncol(Xscale)))
for (k in 1:250) {
  eta <- as.vector(Xscale %*% beta.lasso)
  haz <- as.numeric(exp(eta))
  rsk <- rev(cumsum(rev(haz)))
  P <- outer(haz, rsk, '/')
  P[upper.tri(P)] <- 0
  beta.lasso <- Softfun(beta.lasso + (gam/N)*(t(Xscale) %*% (delta - P %*% delta)),lam*gam)
}
 
round(data.frame(glmnet=coef(mdl.glmnet)[,1],beta.lasso),4)
{% endhighlight %}



{% highlight text %}
##           glmnet beta.lasso
## karno    -0.5414    -0.5414
## diagtime  0.1317     0.1317
## age       0.0000     0.0000
{% endhighlight %}
 
Lastly, we can combine both the Ridge and Lasso models into the single elastic-net framework and once again use proximal gradient descent to update our model:
 
$$
\begin{align*}
&\text{Elnet Cox Proximal-GD update} \\
P(\lambda,\alpha,\bbeta) &= \lambda(\alpha \| \bbeta \|_1 + 0.5(1-\alpha) \| \bbeta \|_{2}^2 ) \\
\bbeta^{(k)} &= S_{\gamma\alpha\lambda}\Bigg(\beta^{(k-1)} - \gamma \frac{\mathcal{p}\ell(\bbeta)}{\partial \bbeta}\Bigg) \\ 
&= S_{\gamma\alpha\lambda}\Bigg(\beta^{(k-1)} + \frac{\gamma}{N}\bX^T(\bdelta - \bP\bdelta) - \gamma\lambda(1-\alpha)\bbeta^{(k-1)} \Bigg) 
\end{align*}
$$
 

{% highlight r %}
# glmnet
lam <- 1/75
alpha <- 1/2
mdl.glmnet <- glmnet(x=Xscale,y=So,family='cox',alpha=alpha,lambda=lam,standardize = F)
 
gam <- 0.1
beta.elnet <- as.matrix(rep(0,ncol(Xscale)))
for (k in 1:250) {
  eta <- as.vector(Xscale %*% beta.elnet)
  haz <- as.numeric(exp(eta))
  rsk <- rev(cumsum(rev(haz)))
  P <- outer(haz, rsk, '/')
  P[upper.tri(P)] <- 0
  beta.elnet <- Softfun(beta.elnet + (gam/N)*(t(Xscale) %*% (delta - P %*% delta) -
                                                gam*lam*(1-alpha)*beta.elnet ),lam*alpha*gam)
}
 
round(data.frame(glmnet=coef(mdl.glmnet)[,1],beta.elnet),4)
{% endhighlight %}



{% highlight text %}
##           glmnet beta.elnet
## karno    -0.5475    -0.5527
## diagtime  0.1428     0.1446
## age       0.0000     0.0000
{% endhighlight %}
 
 
## Summary
 
This post has shown derive the gradient with respect to the Cox PH's partial loss function and perform proximal gradient descent for an elastic net type penalty term. In the next post, I'll outline some extensions to the partial likelihood model including opportunities for multitask learning.
