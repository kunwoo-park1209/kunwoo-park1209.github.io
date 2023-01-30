---
title : "[Data Science] Chebyshev's Theorem"
categories:
  - Data Science
tags:
  - [data science]

toc: true
toc_sticky: true

date: 2023-01-28
last_modified_at: 2023-01-28
---

Chebyshev's Theorem estimates the minimum proportion of observations within a specified number of standard deviations from the mean. Chebyshev’s Theorem applies to every data set and is heavily used by Statisticians, Data Scientists, and Machine Learning Engineers.

## What is the Empirical Rule?

Before we learn about Chebychev's Theorem, it will be helpful to understand the Empirical rule.

The Empirical rule tells you where most values lie in a normal distribution. It states that:

1. around 68% of the data lie within one standard deviation of the mean
2. around 95% of the data lie within two standard deviations of the mean
3. around 99.7% of the data lie within three standard deviations of the mean

If your data follow the normal distribution, that is easy using the Empirical Rule. However, you can never be 100% sure that the data will follow a normal distribution because you may have skewed data. If you don't know the distribution of your data or you know that it doesn't follow the normal distribution, this is where Chebychev's Theorem comes into play.

## What is Chebychev's Theorem?

Chebyshev's Theorem is named after the Russian mathematician Pafnuty Chebyshev (1821-94). It states that a certain proportion of any dataset must fall within a specific range (determined by the standard deviation of the data) around the central mean value. Chebyshev's Theorem can be stated mathematically in the form of an inequality, which is why the Theorem is also known as Chebyshev's Inequality. It applies to any dataset that follows any probability distribution where you can calculate the mean and standard deviation. 

### Equation for Chebyshev's Theorem

There are two ways of presenting Chebyshev's Theorem. One determines how close to the mean the data lie, and the other calculates how far away from the mean they fall:

Let $X$ be a random variable with finite mean $\mu$ and finite standard deviation $\sigma$, and let $k>1$ be any positive number. 

1.  $P(\|X - \mu\| ≥ k\sigma) ≤ 1 / k^2$

    The equation states that the probability that $X$ falls more than $k$ standard deviations away from the mean is at most $1/k^2$. 

2.  $P(\|X - \mu\| ≤ k\sigma) ≥ 1 - 1 / k^2$

    The equation states that the probability that $X$ falls within $k$ standard deviations away from the mean is at least $1-1/k^2$.

The proportion of the data falling within or beyond a certain range of the mean can be determined from the value of $k$ which expresses the size of the range as a count of standard deviations.

| k | Minimum % within k standard deviations | Maximum % with k standard deviations |
| - | - | - |
| $\sqrt2$ | $50$ | $50$ |
| $1.5$ | $56$ | $44$ |
| $2$ | $75$ | $25$ |
| $3$ | $89$ | $11$ |
| $4$ | $94$ | $6$ |
| $5$ | $96$ | $4$ |

For example, if you're interested in a range of three standard deviations around the mean, Chebyshev's Theorem states that at least 89% of the observations fall inside that range and no more than 11% fall outside that range. 

### Observation

1. Chebyshev's Theorem does not provide exact answers but places limits (i.e., minimum and maximum proportions) on the possible proportions because of uncertainties about the data's distribution. The advantage of this theorem is that it applies to 'any' distribution, so long as the mean and standard deviation are defined. The downside is that the estimated proportions are rather crude and for many distributions, the proportion of data close to the mean will be higher than predicted by Chebyshev's inequality.

2. Chebyshev's Theorem implies that it is implausible that a random variable will be far from the mean. 

The image below visually shows you the difference between the Empirical Rule and Chebyshev's Theorem: 
  
![What is Chebychev's Theorem and How Does it Apply to Data Science?](https://www.kdnuggets.com/wp-content/uploads/arya_chebychev_theorem_apply_data_science_1.png)  

## So How Does it Apply to Data Science?

In data science, you use many statistical measures to learn more about the dataset you are using. For example, the mean value is a measure of central tendency of a dataset; however, this does not tell us enough about the dataset. As a data scientist, you will be wondering to learn more about the dispersion of the data. And to figure this out will need to measure the standard deviation, which represents the typical difference between each value and the mean. Together, the mean and standard deviation are a useful description of a dataset as a whole. 

If a random variable has low variance, the observed values will be grouped close to the mean. As shown in the image below, this results in a smaller standard deviation. However, if your data distribution has a large variance, then your observations will naturally be more spread out and further away from the mean. As shown in the image below, this results in a larger standard deviation.

![What is Chebychev's Theorem and How Does it Apply to Data Science?](https://www.kdnuggets.com/wp-content/uploads/arya_chebychev_theorem_apply_data_science_3.png)  

A simple and convenient interpretation of standard deviation is the observation that most data points will typically fall within one standard deviation of the mean. Chebyshev's theorem expands on this and estimates the proportion of data that must fall within two, three, or more deviations of the mean.

Data Scientists and other tech experts will come across datasets that have a large variance. Therefore, they will need more than their usual go-to Empirical rule to help them, and they will have to use Chebyshev's Theorem.

## Conclusion

Chebyshev's Theorem can be applied if you want to learn more about the dispersion of your data points, as long as you have calculated the mean and standard deviation and it does not follow a normal distribution. It will be able to determine the proportion of data falling within a specific range of the mean.

## References

[1] [Kdnuggets - What is Chebychev’s Theorem and How Does it Apply to Data Science?](https://www.kdnuggets.com/2022/11/chebychev-theorem-apply-data-science.html)

[2] [Statistics By Jim - Chebyshev's Theorm in Statistics](https://statisticsbyjim.com/basics/chebyshevs-theorem-in-statistics/)

[3] [What is Chebyshev's Theorem?](https://study.com/learn/lesson/chebyshev-theorem.html)
