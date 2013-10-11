---
title: Some thoughts about Deep Learning
---

Training RBMs
------------

RBMs are a special case of Boltzman machines and Markov random fields. Their graphical model corresponds to that of factor analysis.

RBM's applications:
------------

[*Dimension Reduction*][1]

In this Science paper, a RBM is used to model images where pixels are visible units. A joint configuration $(\mathbf{v}, \mathbf{h})$ of the visible and hidden units has an [energy][2] given by

$$E(\mathbf{v}, \mathbf{h}) = -\sum_{i\in\mathrm{pixels}}b_iv_i - \sum_{j\in\mathrm{features}}b_jh_j-\sum_{i,j}v_ih_jw_{ij}$$

where $v_i$ and $h_j$ are the binary states of pixel $i$ and feature $j$, $b_i$ and $b_j$ are their biases, and $w_{ij}$ is the weight between them. The probability that the model assigns to a visible vector, $\mathbf{v}$ is

$$p(\mathrm{v}) = \sum_{\mathbf{h}\in\mathcal{H}}p(\mathbf{v}, \mathbf{h})=\frac{\sum_h\exp(-E(\mathbf{v},\mathbf{h}))}{\sum_{\mathbf{u}, \mathbf{g}}\exp(-E(\mathbf{u},\mathbf{g}))}$$

where $\mathcal{H}$ is the set of all possible binary hidden vectors. When using linear visible units and binary hidden units, the energy function and update rules are particularly simple if the linear units have Gaussian noise with unit variance. If the variances are not $1$, the energy function becomes:

$$E(\mathbf{v}, \mathbf{h}) = -\sum_{i\in\mathrm{pixels}}\frac{(v_i-b_i)^2}{2\sigma^2_i} - \sum_{j\in\mathrm{features}}b_jh_j-\sum_{i,j}\frac{v_i}{\sigma_i}h_jw_{ij}$$

where $\sigma_i$ is the standard deviation of the Gaussian noise for visible unit $i$. The stochastic update
rule for the hidden units remains the same except that each $v_i$ is divided by $\sigma_i$. The update rule
for visible units $i$ is to sample from a Gaussian with mean $b_i+\sigma_i\sum_jh_jw_{ij}$ and variance $\sigma_i^2$.

The best thing about RBM is that the layer-by-layer learning can be repeated as many times as desired.

[*Classifcation*][3]

[1]: http://www.cs.toronto.edu/~hinton/science.pdf "Geoffrey Hinton and Ruslan Salakhutdinov (2006). Reducing the dimensionality of data with neural networks. Science 28:504–507."

[2]: http://www.pnas.org/content/79/8/2554.full.pdf "J. J. Hopfield, Proc. Natl. Acad. Sci. U.S.A. 79, 2554 (1982)."

[3]: http://machinelearning.org/archive/icml2008/papers/601.pdf "Hugo Larochelle and Yoshua Bengio (2008). Classification using discriminative restricted Boltzmann machines. Proc. 25th Int'l Conf. on Machine Learning."
