.. _tutorial:

Tutorial
========

.. currentmodule:: hmmlearn

``hmmlearn`` implements the Hidden Markov Models (HMMs).
The HMM is a generative probabilistic model, in which a sequence of observable
:math:`\mathbf{X}` variables is generated by a sequence of internal hidden
states :math:`\mathbf{Z}`. The hidden states are not be observed directly.
The transitions between hidden states are assumed to have the form of a
(first-order) Markov chain. They can be specified by the start probability
vector :math:`\boldsymbol{\pi}` and a transition probability matrix
:math:`\mathbf{A}`. The emission probability of an observable can be any
distribution with parameters :math:`\boldsymbol{\theta}` conditioned on the
current hidden state. The HMM is completely determined by
:math:`\boldsymbol{\pi}`, :math:`\mathbf{A}` and :math:`\boldsymbol{\theta}`.

There are three fundamental problems for HMMs:

* Given the model parameters and observed data, estimate the optimal
  sequence of hidden states.

* Given the model parameters and observed data, calculate the likelihood
  of the data.

* Given just the observed data, estimate the model parameters.


The first and the second problem can be solved by the dynamic programming
algorithms known as
the Viterbi algorithm and the Forward-Backward algorithm, respectively.
The last one can be solved by an iterative Expectation-Maximization (EM)
algorithm, known as the Baum-Welch algorithm.

.. topic:: References:

  .. [Rabiner89] Lawrence R. Rabiner "A tutorial on hidden Markov models and
                 selected applications in speech recognition",
                 Proceedings of the IEEE 77.2, pp. 257-286, 1989.
  .. [Bilmes98] Jeff A. Bilmes, "A gentle tutorial of the EM algorithm and its
                application to parameter estimation for Gaussian mixture and
                hidden Markov models.", 1998.

Available models
----------------

.. autosummary::
   :nosignatures:

   hmm.GaussianHMM
   hmm.GMMHMM
   hmm.MultinomialHMM

:ref:`Read on <customizing>` for details on how to implement an HMM with a
custom emission probability.


Building HMM and generating samples
-----------------------------------

You can build an HMM instance by passing the parameters described above to the
constructor. Then, you can generate samples from the HMM by calling
:meth:`~base._BaseHMM.sample`.

::

    >>> import numpy as np
    >>> from hmmlearn import hmm
    >>> np.random.seed(42)

    >>> model = hmm.GaussianHMM(n_components=3, covariance_type="full")
    >>> model.startprob_ = np.array([0.6, 0.3, 0.1])
    >>> model.transmat_ = np.array([[0.7, 0.2, 0.1],
    ...                             [0.3, 0.5, 0.2],
    ...                             [0.3, 0.3, 0.4]])
    >>> model.means_ = np.array([[0.0, 0.0], [3.0, -3.0], [5.0, 10.0]])
    >>> model.covars_ = np.tile(np.identity(2), (3, 1, 1))
    >>> X, Z = model.sample(100)

The transition probability matrix need not to be ergodic. For instance, a
left-right HMM can be defined as follows::

    >>> lr = hmm.GaussianHMM(n_components=3, covariance_type="diag",
    ...                      init_params="cm", params="cmt")
    >>> lr.startprob_ = np.array([1.0, 0.0, 0.0])
    >>> lr.transmat_ = np.array([[0.5, 0.5, 0.0],
    ...                          [0.0, 0.5, 0.5],
    ...                          [0.0, 0.0, 1.0]])

If any of the required parameters are missing, :meth:`~base._BaseHMM.sample`
will raise an exception::

    >>> hmm.GaussianHMM(n_components=3)
    >>> X, Z = model.sample(100)
    Traceback (most recent call last):
        ...
    sklearn.utils.validation.NotFittedError: This GaussianHMM instance is not fitted yet. Call 'fit' with appropriate arguments before using this method.

.. topic:: Fixing parameters

   Each HMM parameter has a character code which can be used to customize its
   initialization and estimation.  EM algorithm needs a starting point to
   proceed, thus prior to training each parameter is assigned a value
   either random or computed from the data. It is possible to hook into this
   process and provide a starting point explicitly. To do so

     1. ensure that the character code for the parameter is missing from
        :attr:`~base._BaseHMM.init_params` and then
     2. set the parameter to the desired value.

   For example, consider an HMM with explicitly initialized transition
   probability matrix

      >>> model = hmm.GaussianHMM(n_components=3, n_iter=100, init_params="mcs")
      >>> model.transmat_ = np.array([[0.7, 0.2, 0.1],
      ...                             [0.3, 0.5, 0.2],
      ...                             [0.3, 0.3, 0.4]])

   A similar trick applies to parameter estimation. If you want to fix some
   parameter at a specific value, remove the corresponding character from
   :attr:`~base._BaseHMM.params` and set the parameter value before training.

.. topic:: Examples:

 * :ref:`sphx_glr_auto_examples_plot_hmm_sampling.py`

Training HMM parameters and inferring the hidden states
-------------------------------------------------------

You can train an HMM by calling the :meth:`~base._BaseHMM.fit` method. The input
is a matrix of concatenated sequences of observations (*aka* samples) along with
the lengths of the sequences (see :ref:`Working with multiple sequences <multiple_sequences>`).

Note, since the EM algorithm is a gradient-based optimization method, it will
generally get stuck in local optima. You should in general try to run ``fit``
with various initializations and select the highest scored model.

The score of the model can be calculated by the :meth:`~base._BaseHMM.score` method.

The inferred optimal hidden states can be obtained by calling
:meth:`~base._BaseHMM.predict` method. The ``predict`` method can be
specified with decoder algorithm. Currently the Viterbi algorithm
(``"viterbi"``), and maximum a posteriori estimation (``"map"``) are supported.

This time, the input is a single sequence of observed values.  Note, the states
in ``remodel`` will have a different order than those in the generating model.

::

    >>> remodel = hmm.GaussianHMM(n_components=3, covariance_type="full", n_iter=100)
    >>> remodel.fit(X)  # doctest: +ELLIPSIS, +NORMALIZE_WHITESPACE
    GaussianHMM(algorithm='viterbi',...
    >>> Z2 = remodel.predict(X)

.. topic:: Monitoring convergence

   The number of EM algorithm iteration is upper bounded by the ``n_iter``
   parameter. The training proceeds until ``n_iter`` steps were performed or the
   change in score is lower than the specified threshold ``tol``. Note, that
   depending on the data EM algorithm may or may not achieve convergence in the
   given number of steps.

   You can use the :attr:`~base._BaseHMM.monitor_` attribute to diagnose
   convergence::

       >>> remodel.monitor_  # doctest: +ELLIPSIS
       ConvergenceMonitor(history=[...],
                 iter=12, n_iter=100, tol=0.01, verbose=False)
       >>> remodel.monitor_.converged
       True

.. _multiple_sequences:

.. topic:: Working with multiple sequences

   All of the examples so far were using a single sequence of observations.
   The input format in the case of multiple sequences is a bit involved and
   is best understood by example.

   Consider two 1D sequences::

       >>> X1 = [[0.5], [1.0], [-1.0], [0.42], [0.24]]
       >>> X2 = [[2.4], [4.2], [0.5], [-0.24]]

   To pass both sequences to :meth:`~base._BaseHMM.fit` or
   :meth:`~base._BaseHMM.predict` first concatenate them into a single array and
   then compute an array of sequence lengths::

       >>> X = np.concatenate([X1, X2])
       >>> lengths = [len(X1), len(X2)]

   Finally just call the desired method with ``X`` and ``lengths``::

       >>> hmm.GaussianHMM(n_components=3).fit(X, lengths)  # doctest: +ELLIPSIS, +NORMALIZE_WHITESPACE
       GaussianHMM(algorithm='viterbi', ...

.. topic:: Examples:

 * :ref:`sphx_glr_auto_examples_plot_hmm_stock_analysis.py`

Saving and loading HMM
----------------------

After traning an HMM can be easily persisted for future use with the standard
:mod:`pickle` module or its more efficient replacement in the :mod:`joblib`
package::

    >>> from sklearn.externals import joblib
    >>> joblib.dump(remodel, "filename.pkl")
    ["filename.pkl"]
    >>> joblib.load("filename.pkl")  # doctest: +ELLIPSIS, +NORMALIZE_WHITESPACE
    GaussianHMM(algorithm='viterbi',...

.. _customizing:

Implementing HMMs with custom emission probabilities
----------------------------------------------------

If you want to implement other emission probability (e.g. Poisson), you have to
subclass :class:`~base._BaseHMM` and override the following methods

.. autosummary::

   base._BaseHMM._init
   base._BaseHMM._check
   base._BaseHMM._generate_sample_from_state
   base._BaseHMM._compute_log_likelihood
   base._BaseHMM._initialize_sufficient_statistics
   base._BaseHMM._accumulate_sufficient_statistics
   base._BaseHMM._do_mstep
