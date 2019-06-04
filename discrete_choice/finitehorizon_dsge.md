---
title: "Dynamic, Finite Horizon, Discrete Choice Models"
author: "John Morehouse"
date: "22 March 2019"
output:
  html_document:
    highlight: haddock
    keep_md: yes
    theme: flatly
    toc: yes
    toc_depth: 4
    toc_float: yes
  pdf_document:
    toc: yes
    toc_depth: '4'
---

*I am grateful to many of the faculty at the University of Oregon whom have helped me understand this. Specifically, I am grateful to [Mark Colas](https://sites.google.com/site/markyaucolas/home) for all of the guidance. Additonally, I would like to thank [Grant Mcdermott](http://grantmcdermott.com/) for his wonderful [course](https://github.com/uo-ec607-2019winter/lectures) that has helped in numerous ways. Any errors are my own.*

# Introduction


I have found there are many resources online for learning about finite horizon discrete dynamic choice models. However, there are few resources for going through an example with code. I intend on posting roughly three set of notes regarding this topic. They will be structured as follows

1. Solving discrete dynamic choice models in partial equilibrium: set-up and examples (this set of notes)

2. Extensions to 1. (expanding the state-space, distributional assumptions, maybe monte-carlo integration)

3. General equilibrium discrete-dynamic choice models: set-up and examples.

#The Setup

I now walk through the set-up of the general problem. This essentially an annotation of [Lee 2005](https://onlinelibrary.wiley.com/doi/full/10.1111/j.0020-6598.2005.00308.x). 

## Decision Problem

Consider an agent chosing between $z$ alternatives, where $z$ is finite. The agent lives for $T$ periods. Let $S(a)$ denote the current set of state variables at age $a$. The decision problem can be stated as:

\begin{align*}
V(\boldsymbol{S}(a),a) &= \max_{ \{d_z(a)\}_{\tau=a}^T } E \left [ \sum_{\tau = a}^T \beta^{\tau -a} u(a) | S(a)  \right ]\\
u(a) &= \sum_z d_z(a) u_z(a)
\end{align*}

where $d_z(t)$ is an indicator that is one if the individual chooses alternative $z$, $u_z$ is the utility associated with that alternative and $\beta$ is the subjective discount rate. In words, this says that the agent needs to pick a sequence of $z$'s that maximize the present-value of lifetime utility. We can write the value function as the maximum over alternative specific value functions:

\begin{align*}
V(\boldsymbol{S}(a),a) = \max_{z \in Z} \{ V_z(\boldsymbol{S}(a), a) \}
\end{align*}
 
where $V_z(\boldsymbol{S}(a), a)$ is the alternative-specific value function. Note that because the agent has a finite life, we can write the alternative specific value function in the following way:


\begin{align*}
V_z((\boldsymbol{S}(a), a) = 
\begin{cases} 
     v(\cdot | S(a)) +  \beta E \left [  V_z(\boldsymbol{S}(a+1), a+1) | \boldsymbol{S}(a) \right  ]  & a < T\\
     &\\
      V_z((\boldsymbol{S}(a), a)& a = T \\
   \end{cases}
\end{align*}



The formulation above of the alternative specific value function hinges on the fact that in the terminal period there is no continuation value. The optimal sequence of location choices is given by:

\[  z_a^* = argmax_z V(S(a)) \]



## The Emax Function

Define the $Emax$ function as the expected continuation value over the stochastic shocks:

\[Emax(S(a))=E\left[ V(S(a) \right ]\\
\int_{-\infty}^{+\infty}\max_z[V_z(S(a))]dF(\varepsilon)  \]

where $F(\varepsilon)$ is the cumulative density function of $\varepsilon$. As mentioned abve the agent has a finite life so we can write:

\begin{align*}
V_z(T) = v_z(S(T))\\
Emax(S(T))= E \left [ \max_z(u_z(S(T))) \right ]
\end{align*}

After we compute $Emax(S(T))$ we can recursively determine the sequence of Emax functions as

\begin{align*}

Emax(S(T-1)) &= E[\max_z V_z(S(T-1))] \\
&=E[\max_z[u_z+\beta Emax(S_z(T))]]
\end{align*}

Note that in the initial step, we have already calculated $Emax(S_z(T))$ for each alternative. Thus we can subsequently determine the entire sequence until we get to the current period.

# An Example

Most posts I have found online have abstracted from a concrete example with code. Here I will walk through an example with an implementation of the alogrithm.


## Model

Consider a model in which an agent is making a decision as to where to live. This model will be similar to the static framework first discussed by Rosen (1984) but we will add dynamics. For simplicity, assume that there are only two locations $z=1$ or $z=2$. These locations can vary by wages, rents and a stochastic shock. We will solve a partial equilibrium version of the model (for now) and hence fix wages and rents.


The agent lives for T periods, and is endowed with an initial location, which we will denote as $z_0$. They care about consumption of a numeaire good and rental prices. If the agent wants to move they must pay a monetary cost[^1], which we will denote $c$. Additionally, each agent receives an idiosyncratic shock, $\varepsilon$ each period. Assume that this shock is distributed extreme-value type 1 (EV1).

## Flow Utility

An optimizing agent will consume the utility maximizing bundle of the consumption good and housing regardless of the location. Hence we can view the agent as choosing a location to maximize utility, taking consumption and housing as given. Suppose the agent has cobb-douglas preferences. Then the static maximization (not conditional on location) problem is:


\[\max_{C,H} C^{\alpha_c}H^{\alpha_h}\exp(\varepsilon)\\
s.t\\
C+R*H \leq W\]

Solving the maximization problem is straightforward. The optimal quantities of consumption and housing are given by:

\[C^* =\alpha_cW, \hspace{.2in} H^* = \frac{\alpha_h}{R}W \]

This yields the indirect utility function

\[v = (\alpha_c+\alpha_h) w-\alpha_h r+ \varepsilon \]

where lower-cased variables are in logs. Note that the agent's location choice does not change the consumption and housing demand functions, rather it impacts the values (wages and rents vary across cities). Location has another impact though when we consider dynamics; if the agent moves, they must pay a cost. Incorporating moving costs, we can write the indirect utility function (henceforth flow utility) for individual $i$ in city $z$ at age $a$ as

\[v_{iza} =  \begin{cases} 
(\alpha_c+\alpha_h) w_z-\alpha_h r_z+\varepsilon_{iza} & \hspace{.2in} \text{if}\hspace{.2in} z_{i,a}= z_{i,a-1}\\
(\alpha_c+\alpha_h)(w_z-c)-\alpha_hr_z +\varepsilon_{iza} & \hspace{.2in} \text{if}\hspace{.2in} z_{i,a}\neq z_{i,a-1}\\
\end{cases}
\]

From here on, lets assume $\alpha_c+\alpha_h=1$. Also note that the state space at age a is $S(a) = \{Z(a-1), \varepsilon(a)  \}$. In general the econometrician cannot observe $\varepsilon(a)$.




# Algorithm

##Overview

I now turn toward simulating the model. In the partial equilibrium set up, this can be broken down into two main steps:

0. Calibrate the model (set wages, rents, etc)

1. Solve for Emax given parameter values, recursively.

  - Note: Here we actually solve for Emax because under the EV1 assumption, this has a closed form.

2. Simulate choices

  - Note: In this step we are *simulating* choices because we are drawing values of $\varepsilon$ each period.



### Stage 1

Given our assumption about $\varepsilon$, Emax has a closed for each period.[^2] Specifically, we can write:

$$Emax = \log \left ( \sum_{z}\exp(\bar{V}_z)   \right ) $$

Where $\bar{V}_m$ is the deterministic part of the value of alternative $z$.  Note that in general $Emax \in \mathbb{R}^z$: there is a continuation value for each alternative. This will become more clear with an example.   

To calculate Emax, we can start in the final period and roll back until the current period. Note that is is useful to denote:

$$v_{iza} = \tilde{v}_{iza}+\varepsilon_{iza} $$


To initialize the alogirithm:


1. **Period t = T**
In the final decision period, 
\begin{align*}
	&Emax(\mathbf{S}(T)) = E [ max_{z}u_z(\mathbf{S}(T))]\\
	\\&=\left [ \begin{array}{c}
	\log(\exp(w_1-\alpha_hr_1)+\exp((w_2-c)-\alpha_hr_2))\\
	\log(\exp((w_1-c)-\alpha_hr_1)+\exp(w_2-\alpha_hr_2))
	\end{array} \right ]
\end{align*}
	
2. **Period t = T-1**
	
We now move onto the next stage: period T-1. We can calculate 
	
\begin{align*}
	&	Emax(\mathbf{S}(T-1)) = E [max_z[ u_z + \beta Emax((\mathbf{S}_z(T))]]]\\
		\\=&\left [
		\begin{array}{c}
		\log(\exp(w_1-\alpha_hr_1+\beta Emax_1)+\exp((w_2-c)-\alpha_hr_2+\beta Emax_2))\\
		\log(\exp((w_1-c)-\alpha_hr_1+\beta Emax_1)+\exp(w_2+\alpha_hr_2+\beta Emax_2))
		\end{array}
		\right ]
\end{align*}

Where $Emax_1$ denotes the first element in the $Emax$ vector from the step above. Now, rinse and repeat this process all the way until we hit the current period.

### Stage 2


Now that we have calculated the emax function for each period we can simulate the agents choices. To do this, we start at the initial period, draw the $\varepsilon$ and solve the agent problem as:

\[ V(S(a)) = max_z [ u_z+ \beta Emax(S(a+1))     ]    \]

This is solved forward, each period. 

# Code Outline


## Load relevant packages

First load relevant packages:


```r
install.packages("pacman",dependencies = T, repos = "cran.us.r-project.org")
```

```
## 
## The downloaded binary packages are in
## 	/var/folders/rl/4v7kdxg946n2p33q6r4zr9hw0000gn/T//RtmpLlzm4s/downloaded_packages
```

```r
library(pacman)
```

```
## Warning: package 'pacman' was built under R version 3.5.2
```

```r
p_load(tidyverse, magrittr,evd)
```


## Model Parameters

Let's define some parameters we won't change much.



```r
#------#
# utility params
#------#
beta = .95
alpha_h = .5
#------#
# other
#------#
sigma_var = 1
T= 100
i=1000
```


## Compute Emax

As stated, we will compute Emax recursively. I initialize the terminal period Emax in each city as the flow utility from each alternative. We then iterate back until the initial period


```r
emax = function(w_1, w_2, r_1,r_2, c){
  #initialize the emax matrix as matrix of zeros
  emax_mat = matrix(NA, nrow=i, ncol=T)  
  #enter last values to kick off alogrithm
  emax_mat[1,T] = log(exp(w_1-alpha_h*r_1)+exp((w_2-c)-alpha_h*r_2)) 
  emax_mat[2,T] = log(exp((w_1-c)-alpha_h*r_1)+exp(w_2-alpha_h*r_2))
  #proceed with the recursive iterations
    for (k in (T-1):1){
      emax_mat[1,k] = log(exp(w_1-alpha_h*r_1+beta*emax_mat[1,k+1])+
                            exp((w_2-c)-alpha_h*r_2+beta*emax_mat[2,k+1]))

      emax_mat[2,k] = log(exp((w_1-c)-alpha_h*r_1+beta*emax_mat[1,k+1])+
                             exp(w_2-alpha_h*r_2+beta*emax_mat[2,k+1]))

    }
    return(emax_mat)
}
#plot it
ggplot()+
geom_line(aes(x= 1:T,emax(2,2,2,2,1)[2,]),col="red")+labs(x="time",y="Emax_2")+
  labs(x= "Time", y= "Emax_2")+theme_minimal()
```

![](finitehorizon_dsge_files/figure-html/emax-1.png)<!-- -->

## Simulate Choices

Now calculate the agent choices. Breakdown of the code:

- Generate idiosyncratic shocks for each city and agent. drawn from same distribution
- Calculate emax for each city
- Generate $i * T$ matrix to hold agent location choice at each point in time
- Initialize agent locations

**Loops**

- outer-loop: over time
- inner-loop: over agents
- remember flow utility in time $t$ depends on where you were last period. Hence we will need an if statement.



```r
agent_choices= function(w_1, w_2, r_1,r_2, c){
  #generate shock matrices
  shocks_1 = replicate(T, rgev(i, loc = 0, scale = 1, shape = 0)) 
  shocks_2 = replicate(T, rgev(i, loc = 0, scale = 1, shape = 0)) 
  #calculate emax sequence
  emax_1 = emax(w_1, w_2, r_1,r_2, c)[1,] 
  emax_2 = emax(w_1, w_2, r_1,r_2, c)[2,]
  choice_mat = matrix(NA, nrow = i, ncol=T)
  #initialize half of agents to city one and half to city two
  choice_mat[1:i/2,1] = 1
  choice_mat[(i/2+1):i,1] = 2
  #k: loops over time
  #j: loops over agents
  #since we only have two cities, I will write out each bit of the state space. Eventually need to loop over states
  for (k in 2:T){
    for (j in 1:i){
      #remember flow utility is conditional on where you were last period.
      if (choice_mat[j,k-1] == 1){
      #which.max is equivalent to argmax for vector entry
      choice_mat[j,k] = which.max(c(w_1-alpha_h*r_1+shocks_1[j,k]+beta*emax_1[k],
                                    (w_2-c)-alpha_h*r_2+shocks_2[j,k]+beta*emax_2[k]))[1]
      } else {
      choice_mat[j,k] = which.max(c((w_1-c)-alpha_h*r_1+shocks_1[j,k]+beta*emax_1[k],
                                    w_2-alpha_h*r_2+shocks_2[j,k]+beta*emax_2[k]))[1]
      }
    }
  }
  return(choice_mat)
}
```

## Visualizing the results

Now, we are probably interested in obtaining a visual of the results. Lets plot the fraction of agents in city 1 in any given period. 


```r
plotter= function(w_1, w_2, r_1,r_2, c){
  choices = agent_choices(w_1, w_2, r_1,r_2, c)
  #get total fraction of agents in city one at each period
  num = subset(choices==1) %>% colSums(choices)
  denom = i
  frac_mat = num/denom
  agent_df = tibble(
  time = 1:T,
  frac_one = frac_mat
  )
  return(ggplot(agent_df)+geom_line(aes(x=time,y=frac_one),col="red")+theme_minimal())
}
```


Great! Now we can plot the results



```r
plotter(2,2,2,2,1)
```

![](finitehorizon_dsge_files/figure-html/code-1.png)<!-- -->



This is excellent. Both cities are equalized, and half of the agents start in city one. The movement we see is being driven entirely by the idiosyncratic preference shock that is received each period. 

# Model Intuition

Now that we have a visualization of the model, we can better understand how certain parameters are affecting the decision problem. For example, suppose city one has a higher wage than city two but the moving cost is higher:

Now, run it and plot it:



```r
#City 1 has higher wage but moving cost is now higher as well
plotter(10,9.8,4,4,7)
```

![](finitehorizon_dsge_files/figure-html/plot2-1.png)<!-- -->

So the higher wage induces more agents to city one but the moving cost slows down how quickly they adjust.


**Note:** Theoretically, the wages and rental prices are in logs. Thus if the moving cost exceeds the wage in a given period, the utility from moving should be negative infinity (and thus the agent wont/cant move). There is nothing in the code currently preventing this. In my next post, I will discuss how to deal with this

# Concluding Remarks

In the next set of notes, we will consider more than two alternatives (and hence add an extra loop to the code: over possible states), as well as altering the distributional assumptions of the shock. As we add more alternatives and head toward general equilibrium, what we are asking of the computer increases substantially. Hence we need to be careful about the code we write- efficiency will become pertinent. I also have this code written in julia for those interested, which I will make available on my github soon. 

Comments are welcome.




 [^1]: See [Bayer et al (2009)](https://www.sciencedirect.com/science/article/pii/S0095069609000035) for a model with moving costs incorporated in a more thoughtful manner.
 
 [^2]: See [Keane, Todd & Wolpin's](https://www.ssc.wisc.edu/~walker/wp/wp-content/uploads/2013/09/KeaneEtalHBLE2011.pdf) text for a discussion.
