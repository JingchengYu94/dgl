.. _guide-minibatch-gpu-sampling:

6.7 Using GPU for Neighborhood Sampling
---------------------------------------

DGL since 0.7 has been supporting GPU-based neighborhood sampling, which has a significant
speed advantage over CPU-based neighborhood sampling.  If you estimate that your graph 
can fit onto GPU and your model does not take a lot of GPU memory, then it is best to
put the graph onto GPU memory and use GPU-based neighbor sampling.

For example, `OGB Products <https://ogb.stanford.edu/docs/nodeprop/#ogbn-products>`_ has
2.4M nodes and 61M edges.  The graph takes less than 1GB since the memory consumption of
a graph depends on the number of edges.  Therefore it is entirely possible to fit the
whole graph onto GPU.

Put the node features onto GPU memory
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

If the node features can also fit onto GPU memory, it is recommended to put them onto GPU
to reduce the time for data transfer from CPU to GPU, which usually becomes a bottleneck
when using GPU for sampling. For exampling, in the above OGB Products, each node has
100-dimensional features and they take less than 1GB memory in total. It is easy to
transfer these features to GPU before training via the following code.

.. code:: python

   # pop the features and labels
   features = g.ndata.pop('features')
   labels = g.ndata.pop('labels')
   # put them onto GPU
   features = features.to('cuda:0')
   labels = labels.to('cuda:0')

If the node features are too large to fit onto GPU memory, :class:`~dgl.contrib.UnifiedTensor`
enables GPU zero-copy access to the features stored on CPU memory and greatly reduces
the time for data transfer from CPU to GPU.


Using GPU-based neighborhood sampling in DGL data loaders
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

One can use GPU-based neighborhood sampling with DGL data loaders via:

* Putting the graph onto GPU.

* Set ``device`` argument to a GPU device.

* Set ``num_workers`` argument to 0, because CUDA does not allow multiple processes
  accessing the same context.

All the other arguments for the :class:`~dgl.dataloading.pytorch.NodeDataLoader` can be
the same as the other user guides and tutorials.

.. code:: python

   g = g.to('cuda:0')
   dataloader = dgl.dataloading.NodeDataLoader(
       g,                                # The graph must be on GPU.
       train_nid,
       sampler,
       device=torch.device('cuda:0'),    # The device argument must be GPU.
       num_workers=0,                    # Number of workers must be 0.
       batch_size=1000,
       drop_last=False,
       shuffle=True)

GPU-based neighbor sampling also works for :class:`~dgl.dataloading.pytorch.EdgeDataLoader` since DGL 0.8.

.. note::

  GPU-based neighbor sampling also works for custom neighborhood samplers as long as
  (1) your sampler is subclassed from :class:`~dgl.dataloading.BlockSampler`, and (2)
  your sampler entirely works on GPU.


Using CUDA UVA-based neighborhood sampling in DGL data loaders
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. note::
   New feature introduced in DGL 0.8.

For the case where the graph is too large to fit onto the GPU memory, we introduce the
CUDA UVA (Unified Virtual Addressing)-based sampling, in which GPUs perform the sampling
on the graph pinned on CPU memory via zero-copy access.
You can enable UVA-based neighborhood sampling in DGL data loaders via:

* Pin the graph to page-locked memory via :func:`dgl.DGLGraph.pin_memory_`.

* Set ``device`` argument to a GPU device.

* Set ``num_workers`` argument to 0, because CUDA does not allow multiple processes
  accessing the same context.

All the other arguments for the :class:`~dgl.dataloading.pytorch.NodeDataLoader` can be
the same as the other user guides and tutorials.
UVA-based neighbor sampling also works for :class:`~dgl.dataloading.pytorch.EdgeDataLoader`.

.. code:: python

   g = g.pin_memory_()
   dataloader = dgl.dataloading.NodeDataLoader(
       g,                                # The graph must be pinned.
       train_nid,
       sampler,
       device=torch.device('cuda:0'),    # The device argument must be GPU.
       num_workers=0,                    # Number of workers must be 0.
       batch_size=1000,
       drop_last=False,
       shuffle=True)

UVA-based sampling is the recommended solution for mini-batch training on large graphs,
especially for multi-GPU training.

.. note::

  To use UVA-based sampling in multi-GPU training, you should first materialize all the
  necessary sparse formats of the graph and copy them to the shared memory explicitly
  before spawning training processes. Then you should pin the shared graph in each training
  process respectively. Refer to our `GraphSAGE example <https://github.com/dmlc/dgl/blob/master/examples/pytorch/graphsage/train_sampling_multi_gpu.py>`_ for more details.


Using GPU-based neighbor sampling with DGL functions
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

You can build your own GPU sampling pipelines with the following functions that support
operating on GPU:

* :func:`dgl.sampling.sample_neighbors`

  * Only has support for uniform sampling; non-uniform sampling can only run on CPU.

Subgraph extraction ops:

* :func:`dgl.node_subgraph`
* :func:`dgl.edge_subgraph`
* :func:`dgl.in_subgraph`
* :func:`dgl.out_subgraph`

Graph transform ops for subgraph construction:

* :func:`dgl.to_block`
* :func:`dgl.compact_graph`
