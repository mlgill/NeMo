NeMo Models
===========

Basics
------

NeMo models contain everything needed to train and reproduce Conversational AI models:

- neural network architectures 
- datasets/data loaders
- data preprocessing/postprocessing
- data augmentors
- optimizers and schedulers
- tokenizers
- language models

NeMo uses `Hydra <https://hydra.cc/>`_ for configuring both NeMo models and the PyTorch Lightning Trainer.

.. note:: Every NeMo model has an example configuration file and training script that can be found `here <https://github.com/NVIDIA/NeMo/tree/v1.0.2/examples>`_.

The end result of using NeMo, `Pytorch Lightning <https://github.com/PyTorchLightning/pytorch-lightning>`_, and Hydra is that NeMo models all have the same look and feel and are also fully compatible with the PyTorch ecosystem. 

Pretrained
----------

NeMo comes with many pretrained models for each of our collections: ASR, NLP, and TTS. Every pretrained NeMo model can be downloaded 
and used with the ``from_pretrained()`` method.

As an example, we can instantiate QuartzNet with the following:

.. code-block:: Python

    import nemo.collections.asr as nemo_asr

    model = nemo_asr.models.EncDecCTCModel.from_pretrained(model_name="QuartzNet15x5Base-En")

To see all available pretrained models for a specific NeMo model, use the ``list_available_models()`` method.

.. code-block:: Python

    nemo_asr.model.EncDecCTCModel.list_available_models()

For detailed information on the available pretrained models, refer to the collections documentation: 

- :ref:`Automatic Speech Recognition (ASR)`
- :ref:`Natural Language Processing (NLP)`
- :ref:`Speech Synthesis (TTS)`

Training
--------

NeMo leverages `PyTorch Lightning <https://www.pytorchlightning.ai/>`_ for model training. PyTorch Lightning lets NeMo decouple the 
conversational AI code from the PyTorch training code. This means that NeMo users can focus on their domain (ASR, NLP, TTS) and 
build complex AI applications without having to rewrite boiler plate code for PyTorch training.

When using PyTorch Lightning, NeMo users can automatically train with:

- multi-GPU/multi-node
- mixed precision
- model checkpointing
- logging
- early stopping
- and more

The two main aspects of the Lightning API are the `LightningModule <https://pytorch-lightning.readthedocs.io/en/stable/common/lightning_module.html#>`_ 
and the `Trainer <https://pytorch-lightning.readthedocs.io/en/stable/common/trainer.html>`_.

PyTorch Lightning ``LightningModule``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Every NeMo model is a ``LightningModule`` which is an ``nn.module``. This means that NeMo models are compatible with the PyTorch 
ecosystem and can be plugged into existing PyTorch workflows.

Creating a NeMo model is similar to any other PyTorch workflow. We start by initializing our model architecture, then define the forward pass:

.. code-block:: python

    class TextClassificationModel(NLPModel, Exportable):
        ...
        def __init__(self, cfg: DictConfig, trainer: Trainer = None):
            """Initializes the BERTTextClassifier model."""
            ...
            super().__init__(cfg=cfg, trainer=trainer)

            # instantiate a BERT based encoder
            self.bert_model = get_lm_model(
                pretrained_model_name=cfg.language_model.pretrained_model_name,
                config_file=cfg.language_model.config_file,
                config_dict=cfg.language_model.config,
                checkpoint_file=cfg.language_model.lm_checkpoint,
                vocab_file=cfg.tokenizer.vocab_file,
            )

            # instantiate the FFN for classification
            self.classifier = SequenceClassifier(
                hidden_size=self.bert_model.config.hidden_size,
                num_classes=cfg.dataset.num_classes,
                num_layers=cfg.classifier_head.num_output_layers,
                activation='relu',
                log_softmax=False,
                dropout=cfg.classifier_head.fc_dropout,
                use_transformer_init=True,
                idx_conditioned_on=0,
            )

.. code-block:: python

        def forward(self, input_ids, token_type_ids, attention_mask):
            """
            No special modification required for Lightning, define it as you normally would
            in the `nn.Module` in vanilla PyTorch.
            """
            hidden_states = self.bert_model(
                input_ids=input_ids, token_type_ids=token_type_ids, attention_mask=attention_mask
            )
            logits = self.classifier(hidden_states=hidden_states)
            return logits

The ``LightningModule`` organizes PyTorch code so that across all NeMo models we have a similar look and feel.
For example, the training logic can be found in ``training_step``:

.. code-block:: python

    def training_step(self, batch, batch_idx):
        """
        Lightning calls this inside the training loop with the data from the training dataloader
        passed in as `batch`.
        """
        # forward pass
        input_ids, input_type_ids, input_mask, labels = batch
        logits = self.forward(input_ids=input_ids, token_type_ids=input_type_ids, attention_mask=input_mask)

        train_loss = self.loss(logits=logits, labels=labels)

        lr = self._optimizer.param_groups[0]['lr']

        self.log('train_loss', train_loss)
        self.log('lr', lr, prog_bar=True)

        return {
            'loss': train_loss,
            'lr': lr,
        }

While validation logic can be found in ``validation_step``:

.. code-block:: python

    def validation_step(self, batch, batch_idx):
        """
        Lightning calls this inside the validation loop with the data from the validation dataloader
        passed in as `batch`.
        """
        if self.testing:
            prefix = 'test'
        else:
            prefix = 'val'

        input_ids, input_type_ids, input_mask, labels = batch
        logits = self.forward(input_ids=input_ids, token_type_ids=input_type_ids, attention_mask=input_mask)

        val_loss = self.loss(logits=logits, labels=labels)

        preds = torch.argmax(logits, axis=-1)

        tp, fn, fp, _ = self.classification_report(preds, labels)

        return {'val_loss': val_loss, 'tp': tp, 'fn': fn, 'fp': fp}

PyTorch Lightning then handles all of the boiler plate code needed for training. Virtually any aspect of training can be customized 
via PyTorch Lightning `hooks <https://pytorch-lightning.readthedocs.io/en/stable/common/lightning_module.html#hooks>`_, 
`Plugins <https://pytorch-lightning.readthedocs.io/en/stable/extensions/plugins.html>`_, 
`callbacks <https://pytorch-lightning.readthedocs.io/en/stable/extensions/callbacks.html>`_, or by overriding `methods <https://pytorch-lightning.readthedocs.io/en/stable/common/lightning_module.html#methods>`_. 

For more domain-specific information, see:

- :ref:`Automatic Speech Recognition (ASR)`
- :ref:`Natural Language Processing (NLP)`
- :ref:`Speech Synthesis (TTS)`

PyTorch Lightning Trainer
~~~~~~~~~~~~~~~~~~~~~~~~~

Since every NeMo model is a ``LightningModule``, we can automatically take advantage of the PyTorch Lightning ``Trainer``. Every NeMo 
`example <https://github.com/NVIDIA/NeMo/tree/v1.0.2/examples>`_ training script uses the ``Trainer`` object to fit the model.

First, instantiate the model and trainer, then call ``.fit``:

.. code-block:: python
    
    # We first instantiate the trainer based on the model configuration.
    # See the model configuration documentation for details.    
    trainer = pl.Trainer(**cfg.trainer)

    # Then pass the model configuration and trainer object into the NeMo model
    model = TextClassificationModel(cfg.model, trainer=trainer)

    # Now we can train with by calling .fit
    trainer.fit(model)

    # Or we can run the test loop on test data by calling
    trainer.test(model=model)

All `trainer flags <https://pytorch-lightning.readthedocs.io/en/stable/common/trainer.html#trainer-flags>`_ can be set from from the 
NeMo configuration. 
    

Configuration
-------------

Hydra is an open-source Python framework that simplifies configuration for complex applications that must bring together many different 
software libraries. Conversational AI model training is a great example of such an application. To train a conversational AI model, we 
must be able to configure:

- neural network architectures
- training and optimization algorithms 
- data pre/post processing
- data augmentation
- experiment logging/visualization
- model checkpointing   

For an introduction to using Hydra, refer to the `Hydra Tutorials <https://hydra.cc/docs/tutorials/intro>`_.

With Hydra, we can configure everything needed for NeMo with three interfaces:

- Command Line (CLI) 
- Configuration Files (YAML)
- Dataclasses (Python)

YAML
~~~~

NeMo provides YAML configuration files for all of our `example <https://github.com/NVIDIA/NeMo/tree/v1.0.2/examples>`_ training scripts.
YAML files make it easy to experiment with different model and training configurations.

Every NeMo example YAML has the same underlying configuration structure:

- trainer
- exp_manager
- model

Model configuration always contain ``train_ds``, ``validation_ds``, ``test_ds``, and ``optim``.  Model architectures vary across 
domains, therefore, refer to the ASR, NLP, and TTS Collections documentation for more detailed information on Model architecture configuration.

A NeMo configuration file should look similar to the following:

.. code-block:: yaml

    # PyTorch Lightning Trainer configuration
    # any argument of the Trainer object can be set here
    trainer:
        gpus: 1 # number of gpus per node
        num_nodes: 1 # number of nodes
        max_epochs: 10 # how many training epochs to run
        val_check_interval: 1.0 # run validation after every epoch

    # Experiment logging configuration
    exp_manager:
        exp_dir: /path/to/my/nemo/experiments
        name: name_of_my_experiment
        create_tensorboard_logger: True
        create_wandb_logger: True

    # Model configuration
    # model network architecture, train/val/test datasets, data augmentation, and optimization
    model:
        train_ds:
            manifest_filepath: /path/to/my/train/manifest.json
            batch_size: 256
            shuffle: True
        validation_ds:
            manifest_filepath: /path/to/my/validation/manifest.json
            batch_size: 32
            shuffle: False
        test_ds:
            manifest_filepath: /path/to/my/test/manifest.json
            batch_size: 32
            shuffle: False
        optim:
            name: novograd
            lr: .01
            betas: [0.8, 0.5]
            weight_decay: 0.001
        # network architecture can vary greatly depending on the domain
        encoder:
            ...
        decoder:
            ...

More specific details about configuration files for each collection can be found on the following pages:

:ref:`NeMo ASR Configuration Files`
        
CLI
~~~

With NeMo and Hydra, every aspect of model training can be modified from the command-line. This is extremely helpful for running lots 
of experiments on compute clusters or for quickly testing parameters while developing.

All NeMo `examples <https://github.com/NVIDIA/NeMo/tree/v1.0.2/examples>`_ come with instructions on how to
run the training/inference script from the command-line (see `here <https://github.com/NVIDIA/NeMo/blob/4e9da75f021fe23c9f49404cd2e7da4597cb5879/examples/asr/asr_ctc/speech_to_text_ctc.py#L24>`_
for an example).

With Hydra, arguments are set using the ``=`` operator:

.. code-block:: bash

    python examples/asr/asr_ctc/speech_to_text_ctc.py \
        model.train_ds.manifest_filepath=/path/to/my/train/manifest.json \
        model.validation_ds.manifest_filepath=/path/to/my/validation/manifest.json \
        trainer.gpus=2 \
        trainer.max_epochs=50

We can use the ``+`` operator to add arguments from the CLI:

.. code-block:: bash

    python examples/asr/asr_ctc/speech_to_text_ctc.py \
        model.train_ds.manifest_filepath=/path/to/my/train/manifest.json \
        model.validation_ds.manifest_filepath=/path/to/my/validation/manifest.json \
        trainer.gpus=2 \
        trainer.max_epochs=50 \
        +trainer.fast_dev_run=true

We can use the ``~`` operator to remove configurations:

.. code-block:: bash

    python examples/asr/asr_ctc/speech_to_text_ctc.py \
        model.train_ds.manifest_filepath=/path/to/my/train/manifest.json \
        model.validation_ds.manifest_filepath=/path/to/my/validation/manifest.json \
        ~model.test_ds \
        trainer.gpus=2 \
        trainer.max_epochs=50 \
        +trainer.fast_dev_run=true

We can specify configuration files using the ``--config-path`` and ``--config-name`` flags:

.. code-block:: bash

    python examples/asr/asr_ctc/speech_to_text_ctc.py \
        --config-path=conf/quartznet \
        --config-name=quartznet_15x5 \
        model.train_ds.manifest_filepath=/path/to/my/train/manifest.json \
        model.validation_ds.manifest_filepath=/path/to/my/validation/manifest.json \
        ~model.test_ds \
        trainer.gpus=2 \
        trainer.max_epochs=50 \
        +trainer.fast_dev_run=true

Dataclasses
~~~~~~~~~~~

Dataclasses allow NeMo to ship model configurations as part of the NeMo library and also enables pure Python configuration of NeMo models. 
With Hydra, dataclasses can be used to create `structured configs <https://hydra.cc/docs/tutorials/structured_config/intro>`_ for the conversational AI application. 

As an example, refer to the code block below for an *Attenion is All You Need* machine translation model. The model configuration can 
be instantiated and modified like any Python `Dataclass <https://docs.python.org/3/library/dataclasses.html>`_.

.. code-block:: Python

    from nemo.collections.nlp.models.machine_translation.mt_enc_dec_config import AAYNBaseConfig

    cfg = AAYNBaseConfig()

    # modify the number of layers in the encoder
    cfg.encoder.num_layers = 8

    # modify the training batch size
    cfg.train_ds.tokens_in_batch = 8192

.. note:: Configuration with Hydra always has the following precedence CLI > YAML > Dataclass

.. _optimization-label:

Optimization
------------

Optimizers and learning rate schedules are configurable across all NeMo models and have their own namespace. Here is a sample YAML 
configuration for a Novograd optimizer with Cosine Annealing learning rate schedule.

.. code-block:: yaml

    optim:
        name: novograd
        lr: 0.01
    
        # optimizer arguments
        betas: [0.8, 0.25]
        weight_decay: 0.001
    
        # scheduler setup
        sched:
        name: CosineAnnealing
    
        # Optional arguments
        max_steps: null # computed at runtime or explicitly set here
        monitor: val_loss
        reduce_on_plateau: false
    
        # scheduler config override
        warmup_steps: 1000
        warmup_ratio: null
        min_lr: 1e-9:

.. note:: `NeMo Examples <https://github.com/NVIDIA/NeMo/tree/v1.0.2/examples>`_ has optimizer and scheduler configurations for
every NeMo model. 

Optimizers can be configured from the CLI as well:

.. code-block:: bash

    python examples/asr/asr_ctc/speech_to_text_ctc.py \
        --config-path=conf/quartznet \
        --config-name=quartznet_15x5 \
        ...
        # train with the adam optimizer
        model.optim=adam \
        # change the learning rate
        model.optim.lr=.0004 \
        # modify betas 
        model.optim.betas=[.8, .5]

.. _optimizers-label:

Optimizers
~~~~~~~~~~

``name`` corresponds to the lowercase name of the optimizer. To view a list of available optimizers, run:

.. code-block:: Python

    from nemo.core.optim.optimizers import AVAILABLE_OPTIMIZERS

    for name, opt in AVAILABLE_OPTIMIZERS.items():
        print(f'name: {name}, opt: {opt}')

.. code-block:: bash

    name: sgd opt: <class 'torch.optim.sgd.SGD'>
    name: adam opt: <class 'torch.optim.adam.Adam'>
    name: adamw opt: <class 'torch.optim.adamw.AdamW'>
    name: adadelta opt: <class 'torch.optim.adadelta.Adadelta'>
    name: adamax opt: <class 'torch.optim.adamax.Adamax'>
    name: adagrad opt: <class 'torch.optim.adagrad.Adagrad'>
    name: rmsprop opt: <class 'torch.optim.rmsprop.RMSprop'>
    name: rprop opt: <class 'torch.optim.rprop.Rprop'>
    name: novograd opt: <class 'nemo.core.optim.novograd.Novograd'>

Optimizer Params
~~~~~~~~~~~~~~~~

Optimizer params can vary between optimizers but the ``lr`` param is required for all optimizers. To see the available params for an 
optimizer, we can look at its corresponding dataclass.

.. code-block:: python

    from nemo.core.config.optimizers import NovogradParams

    print(NovogradParams())

.. code-block:: bash

    NovogradParams(lr='???', betas=(0.95, 0.98), eps=1e-08, weight_decay=0, grad_averaging=False, amsgrad=False, luc=False, luc_trust=0.001, luc_eps=1e-08)

``'???'`` indicates that the lr argument is required.

Register Optimizer
~~~~~~~~~~~~~~~~~~

To register a new optimizer to be used with NeMo, run:

.. autofunction:: nemo.core.optim.optimizers.register_optimizer

.. _learning-rate-schedulers-label:

Learning Rate Schedulers
~~~~~~~~~~~~~~~~~~~~~~~~

Learning rate schedulers can be optionally configured under the ``optim.sched`` namespace.

``name`` corresponds to the name of the learning rate schedule. To view a list of available schedulers, run: 
    
.. code-block:: Python

    from nemo.core.optim.lr_scheduler import AVAILABLE_SCHEDULERS

    for name, opt in AVAILABLE_SCHEDULERS.items():
        print(f'name: {name}, schedule: {opt}')

.. code-block:: bash

    name: WarmupPolicy, schedule: <class 'nemo.core.optim.lr_scheduler.WarmupPolicy'>
    name: WarmupHoldPolicy, schedule: <class 'nemo.core.optim.lr_scheduler.WarmupHoldPolicy'>
    name: SquareAnnealing, schedule: <class 'nemo.core.optim.lr_scheduler.SquareAnnealing'>
    name: CosineAnnealing, schedule: <class 'nemo.core.optim.lr_scheduler.CosineAnnealing'>
    name: NoamAnnealing, schedule: <class 'nemo.core.optim.lr_scheduler.NoamAnnealing'>
    name: WarmupAnnealing, schedule: <class 'nemo.core.optim.lr_scheduler.WarmupAnnealing'>
    name: InverseSquareRootAnnealing, schedule: <class 'nemo.core.optim.lr_scheduler.InverseSquareRootAnnealing'>
    name: SquareRootAnnealing, schedule: <class 'nemo.core.optim.lr_scheduler.SquareRootAnnealing'>
    name: PolynomialDecayAnnealing, schedule: <class 'nemo.core.optim.lr_scheduler.PolynomialDecayAnnealing'>
    name: PolynomialHoldDecayAnnealing, schedule: <class 'nemo.core.optim.lr_scheduler.PolynomialHoldDecayAnnealing'>
    name: StepLR, schedule: <class 'torch.optim.lr_scheduler.StepLR'>
    name: ExponentialLR, schedule: <class 'torch.optim.lr_scheduler.ExponentialLR'>
    name: ReduceLROnPlateau, schedule: <class 'torch.optim.lr_scheduler.ReduceLROnPlateau'>
    name: CyclicLR, schedule: <class 'torch.optim.lr_scheduler.CyclicLR'>

Scheduler Params
~~~~~~~~~~~~~~~~

To see the available params for a scheduler, we can look at its corresponding dataclass:

.. code-block:: Python

    from nemo.core.config.schedulers import CosineAnnealingParams

    print(CosineAnnealingParams())

.. code-block:: bash

    CosineAnnealingParams(last_epoch=-1, warmup_steps=None, warmup_ratio=None, min_lr=0.0)

Register scheduler
~~~~~~~~~~~~~~~~~~

To register a new scheduler to be used with NeMo, run:

.. autofunction:: nemo.core.optim.lr_scheduler.register_scheduler

Save and Restore
----------------

NeMo models all come with ``.save_to`` and ``.restore_from`` methods.  

Save
~~~~

To save a NeMo model, run:

.. code-block:: Python

    model.save_to('/path/to/model.nemo')

Everything needed to use the trained model is packaged and saved in the ``.nemo`` file. For example, in the NLP domain, ``.nemo`` files 
include the necessary tokenizer models and/or vocabulary files, etc.

.. note:: A ``.nemo`` file is simply an archive like any other ``.tar`` file.

Restore
~~~~~~~

To restore a NeMo model, run:

.. code-block:: Python

    model.restore_from('/path/to/model.nemo')

When using the PyTorch Lightning Trainer, a PyTorch Lightning checkpoint is created. These are mainly used within NeMo to auto-resume 
training. Since NeMo models are ``LightningModules``, the PyTorch Lightning method ``load_from_checkpoint`` is available. Note that 
``load_from_checkpoint`` won't necessarily work out-of-the-box for all models as some models require more artifacts than just the 
checkpoint to be restored. For these models, the user will have to override ``load_from_checkpoint`` if they want to use it.

It's highly recommended to use ``restore_from`` to load NeMo models.

Register Artifacts
------------------

Conversational AI models can be complicated to restore as more information is needed than just the checkpoint weights in order to use the model.
NeMo models can save additional artifacts in the .nemo file by calling ``.register_artifact``.
When restoring NeMo models using ``.restore_from`` or ``.from_pretrained``, any artifacts that were registered will be available automatically.

As an example, consider an NLP model that requires a trained tokenizer model. 
The tokenizer model file can be automatically added to the .nemo file with the following:

.. code-block:: python

    self.encoder_tokenizer = get_nmt_tokenizer(
        ...
        tokenizer_model=self.register_artifact(config_path='encoder_tokenizer.tokenizer_model',
                                               src='/path/to/tokenizer.model',
                                               verify_src_exists=True),
    )

By default, ``.register_artifact`` will always return a path. If the model is being restored from a .nemo file, 
then that path will be to the artifact in the .nemo file. Otherwise, ``.register_artifact`` will return the local path specified by the user.

``config_path`` is the artifact key. It usually corresponds to a model configuration but does not have to.
The model config that is packaged with the .nemo file will be updated according to the ``config_path`` key.
In the above example, the model config will have 

.. code-block:: YAML

    encoder_tokenizer:
        ...
        tokenizer_model: nemo:4978b28103264263a03439aaa6560e5e_tokenizer.model

``src`` is the path to the artifact and the base-name of the path will be used when packaging the artifact in the .nemo file.
Each artifact will have a hash prepended to the basename of ``src`` in the .nemo file. This is to prevent collisions with basenames 
base-names that are identical (say when there are two or more tokenizers, both called `tokenizer.model`).
The resulting .nemo file will then have the following file:

.. code-block:: bash

    4978b28103264263a03439aaa6560e5e_tokenizer.model

If ``verify_src_exists`` is set to ``False``, then the artifact is optional. This means that ``.register_artifact`` will return ``None`` 
if the ``src`` cannot be found. 

.. _exp-manager-label:

Experiment Manager
==================

NeMo's Experiment Manager leverages PyTorch Lightning for model checkpointing, TensorBoard Logging, and Weights and Biases logging. The 
Experiment Manager is included by default in all NeMo example scripts.

To use the experiment manager simply call :class:`~nemo.utils.exp_manager.exp_manager` and pass in the PyTorch Lightning ``Trainer``.

.. code-block:: python

    exp_manager(trainer, cfg.get("exp_manager", None))

And is configurable via YAML with Hydra.

.. code-block:: bash

    exp_manager:
        exp_dir: /path/to/my/experiments
        name: my_experiment_name
        create_tensorboard_logger: True
        create_checkpoint_callback: True

Optionally, launch TensorBoard to view the training results in ``./nemo_experiments`` (by default).

.. code-block:: bash

    tensorboard --bind_all --logdir nemo_experiments

..

If ``create_checkpoint_callback`` is set to ``True``, then NeMo automatically creates checkpoints during training
using PyTorch Lightning's `ModelCheckpoint <https://pytorch-lightning.readthedocs.io/en/stable/extensions/generated/pytorch_lightning.callbacks.ModelCheckpoint.html#pytorch_lightning.callbacks.ModelCheckpoint>`_.
We can configure the ``ModelCheckpoint`` via YAML or CLI.

.. code-block:: yaml

    exp_manager:
        ...
        # configure the PyTorch Lightning ModelCheckpoint using checkpoint_call_back_params
        # any ModelCheckpoint argument can be set here

        # save the best checkpoints based on this metric
        checkpoint_callback_params.monitor=val_loss 
        
        # choose how many total checkpoints to save
        checkpoint_callback_params.save_top_k=5

We can auto-resume training as well by configuring the ``exp_manager``. Being able to auto-resume is important when doing long training 
runs that are premptible or may be shut down before the training procedure has completed. To auto-resume training, set the following 
via YAML or CLI:

.. code-block:: yaml

    exp_manager:
        ...
        # resume training if checkpoints already exist
        resume_if_exists: True

        # to start training with no existing checkpoints
        resume_ignore_no_checkpoint: True

        # by default experiments will be versioned by datetime
        # we can set our own version with
        exp_manager.version: my_experiment_version

Neural Modules
==============

NeMo is built around Neural Modules, conceptual blocks of neural networks that take typed inputs and produce typed outputs. Such 
modules typically represent data layers, encoders, decoders, language models, loss functions, or methods of combining activations.
NeMo makes it easy to combine and re-use these building blocks while providing a level of semantic correctness checking via its neural 
type system.

.. note:: *All Neural Modules inherit from ``torch.nn.Module`` and are therefore compatible with the PyTorch ecosystem.*

There are 3 types on Neural Modules:

    - Regular modules
    - Dataset/IterableDataset
    - Losses

Every Neural Module in NeMo must inherit from `nemo.core.classes.module.NeuralModule` class.

.. autoclass:: nemo.core.classes.module.NeuralModule

Every Neural Modules inherits the ``nemo.core.classes.common.Typing`` interface and needs to define neural types for its inputs and outputs.
This is done by defining two properties: ``input_types`` and ``output_types``. Each property should return an ordered dictionary of 
"port name"->"port neural type" pairs. Here is the example from :class:`~nemo.collections.asr.modules.ConvASREncoder` class:

.. code-block:: python

    @property
    def input_types(self):
        return OrderedDict(
            {
                "audio_signal": NeuralType(('B', 'D', 'T'), SpectrogramType()),
                "length": NeuralType(tuple('B'), LengthsType()),
            }
        )

    @property
    def output_types(self):
        return OrderedDict(
            {
                "outputs": NeuralType(('B', 'D', 'T'), AcousticEncodedRepresentation()),
                "encoded_lengths": NeuralType(tuple('B'), LengthsType()),
            }
        )

    @typecheck()
    def forward(self, audio_signal, length=None):
        ...

The code snippet above means that ``nemo.collections.asr.modules.conv_asr.ConvASREncoder`` expects two arguments:
    * First one, named ``audio_signal`` of shape ``[batch, dimension, time]`` with elements representing spectrogram values.
    * Second one, named ``length`` of shape ``[batch]`` with elements representing lengths of corresponding signals.

It also means that ``.forward(...)`` and ``__call__(...)`` methods each produce two outputs:
    * First one, of shape ``[batch, dimension, time]`` but with elements representing encoded representation (``AcousticEncodedRepresentation`` class).
    * Second one, of shape ``[batch]``, corresponding to their lengths.

.. tip:: It is a good practice to define types and add ``@typecheck()`` decorator to your ``.forward()`` method after your module is ready for use by others.

.. note:: The outputs of ``.forward(...)`` method will always be of type ``torch.Tensor`` or container of tensors and will work with any other Pytorch code. The type information is attached to every output tensor.
If tensors without types is passed to your module, it will not fail, however the types will not be checked. Thus, it is recommended to define input/output types for all your modules, starting with
data layers and add ``@typecheck()`` decorator to them.

.. note:: To temporarily disable typechecking, you can enclose your code in ```with typecheck.disable_checks():``` statement.

Neural Types
============

Motivation
----------

Neural Types describe the semantics, axis order, and dimensions of a tensor. The purpose of this type system is to catch semantic and 
dimensionality errors during model creation and facilitate module re-use.

.. image:: whyntypes.gif
  :width: 900
  :alt: Neural Types Motivation

``NeuralType`` class
--------------------

Neural Types perform semantic checks for modules and models inputs/outputs. They contain information about:

    - Semantics of what is stored in the tensors. For example, logits, logprobs, audiosignal, embeddings, etc.
    - Axes layout, semantic and (optionally) dimensionality. For example: ``[Batch, Time, Channel]``

Types are implemented in ``nemo.core.neural_types.NeuralType`` class. When you instantiate an instance of this class, you
are expected to include both *axes* information and *element type* information.

.. autoclass:: nemo.core.neural_types.NeuralType

Type Comparison Results
-----------------------

When comparing two neural types, the following comparison results are generated.

.. autoclass:: nemo.core.neural_types.NeuralTypeComparisonResult

Examples
--------

Long vs short notation
~~~~~~~~~~~~~~~~~~~~~~

NeMo's ``NeuralType`` class allows you to express axis semantics information in long and short form. Consider these two equivalent types. Both encoder 3 dimensional tensors and both contain elements of type ``AcousticEncodedRepresentation`` (this type is a typical output of ASR encoders).

.. code-block:: python

    long_version = NeuralType(
            axes=(AxisType(AxisKind.Batch, None), AxisType(AxisKind.Dimension, None), AxisType(AxisKind.Time, None)),
            elements_type=AcousticEncodedRepresentation(),
        )
    short_version = NeuralType(('B', 'D', 'T'), AcousticEncodedRepresentation())
    assert long_version.compare(short_version) == NeuralTypeComparisonResult.SAME

Transpose same
~~~~~~~~~~~~~~

Often it is useful to know if a simple transposition will solve type incompatibility. This is the case if the comparison result of two types equals ``nemo.core.neural_types.NeuralTypeComparisonResult.TRANSPOSE_SAME``.

.. code-block:: python

    type1 = NeuralType(axes=('B', 'T', 'C'))
    type2 = NeuralType(axes=('T', 'B', 'C'))
    assert type1.compare(type2) == NeuralTypeComparisonResult.TRANSPOSE_SAME
    assert type2.compare(type1) == NeuralTypeComparisonResult.TRANSPOSE_SAME

Note that in this example, we dropped ``elements_type`` argument of ``NeuralType`` constructor. If not supplied, the element type is ``VoidType``.

``VoidType`` for elements
~~~~~~~~~~~~~~~~~~~~~~~~~

Sometimes it is useful to express that elements' types don't matter but axes layout do. ``VoidType`` for elements can be used to express this.

.. note:: ``VoidType`` is compatible with every other elements' type but not the other way around. See the following code snippet below for details.

.. code-block:: python

        btc_spctr = NeuralType(('B', 'T', 'C'), SpectrogramType())
        btc_spct_bad = NeuralType(('B', 'T'), SpectrogramType())
        # Note the VoidType for elements here
        btc_void = NeuralType(('B', 'T', 'C'), VoidType())

        # This is true because VoidType is compatible with every other element type (SpectrogramType in this case)
        # And axes layout between btc_void and btc_spctr is the same
        assert btc_void.compare(btc_spctr) == NeuralTypeComparisonResult.SAME
        # These two types are incompatible because even though VoidType is used for elements on one side,
        # the axes layout is different
        assert btc_void.compare(btc_spct_bad) == NeuralTypeComparisonResult.INCOMPATIBLE
        # Note that even though VoidType is compatible with every other type, other types are not compatible with VoidType!
        # It is one-way compatibility
        assert btc_spctr.compare(btc_void) == NeuralTypeComparisonResult.INCOMPATIBLE

Element type inheritance
~~~~~~~~~~~~~~~~~~~~~~~~

Neural types in NeMo support Python inheritance between element types. Consider an example where you want to develop a Neural Module which performs data augmentation for all kinds of spectrograms.
In ASR, two types of spectrograms are frequently used: mel and mfcc. To express this, we will create 3 classes to express
element's types: ``SpectrogramType``, ``MelSpectrogramType(SpectrogramType)``, ``MFCCSpectrogramType(SpectrogramType)``.

.. code-block:: python

        input = NeuralType(('B', 'D', 'T'), SpectrogramType())
        out1 = NeuralType(('B', 'D', 'T'), MelSpectrogramType())
        out2 = NeuralType(('B', 'D', 'T'), MFCCSpectrogramType())

        # MelSpectrogram and MFCCSpectrogram are not interchangeable.
        assert out1.compare(out2) == NeuralTypeComparisonResult.INCOMPATIBLE
        assert out2.compare(out1) == NeuralTypeComparisonResult.INCOMPATIBLE
        # Type comparison detects that MFCC/MelSpectrogramType is a kind of SpectrogramType and can be accepted.
        assert input.compare(out1) == NeuralTypeComparisonResult.GREATER
        assert input.compare(out2) == NeuralTypeComparisonResult.GREATER

Custom element types
~~~~~~~~~~~~~~~~~~~~

It is possible to create user-defined element types to express the semantics of elements in your tensors. To do so, the user will need to inherit and implement abstract methods of the ``nemo.core.neural_types.elements.ElementType`` class

.. autoclass:: nemo.core.neural_types.elements.ElementType

Note that element types can be parametrized. Consider this example where it distinguishes between audio sampled at 8Khz and 16Khz.

.. code-block:: python

    audio16K = NeuralType(axes=('B', 'T'), elements_type=AudioSignal(16000))
    audio8K = NeuralType(axes=('B', 'T'), elements_type=AudioSignal(8000))

    assert audio8K.compare(audio16K) == NeuralTypeComparisonResult.SAME_TYPE_INCOMPATIBLE_PARAMS
    assert audio16K.compare(audio8K) == NeuralTypeComparisonResult.SAME_TYPE_INCOMPATIBLE_PARAMS

Enforcing dimensions
~~~~~~~~~~~~~~~~~~~~

In addition to specifying tensor layout and elements' semantics, neural types also allow you to enforce tensor dimensions.
The user will have to use long notations to specify dimensions. Short notations only allows you to specify axes semantics and assumes
arbitrary dimensions.

.. code-block:: python

        type1 = NeuralType(
        (AxisType(AxisKind.Batch, 64), AxisType(AxisKind.Time, 10), AxisType(AxisKind.Dimension, 128)),
        SpectrogramType(),
        )
        type2 = NeuralType(('B', 'T', 'C'), SpectrogramType())

        # type2 will accept elements of type1 because their axes semantics match and type2 does not care about dimensions
        assert type2.compare(type1), NeuralTypeComparisonResult.SAME
        # type1 will not accept elements of type2 because it need dimensions to match strictly.
        assert type1.compare(type2), NeuralTypeComparisonResult.DIM_INCOMPATIBLE

Generic Axis kind
~~~~~~~~~~~~~~~~~

Sometimes (especially in the case of loss modules) it is useful to be able to specify a "generic" axis kind which will make it
compatible with any other kind of axis. This is easy to express with Neural Types by using ``nemo.core.neural_types.axes.AxisKind.Any`` for axes.

.. code-block:: python

        type1 = NeuralType(('B', 'Any', 'Any'), SpectrogramType())
        type2 = NeuralType(('B', 'T', 'C'), SpectrogramType())
        type3 = NeuralType(('B', 'C', 'T'), SpectrogramType())

        # type1 will accept elements of type2 and type3 because it only cares about element kind (SpectrogramType)
        # number of axes (3) and that first one corresponds to batch
        assert type1.compare(type2) == NeuralTypeComparisonResult.SAME
        assert type1.compare(type3) == NeuralTypeComparisonResult.INCOMPATIBLE

Container types
~~~~~~~~~~~~~~~

The NeMo-type system understands Python containers (lists). If your module returns a nested list of typed tensors, the way to express it is by
using Python list notation and Neural Types together when defining your input/output types.

The example below shows how to express that your module returns single output ("out") which is list of lists of two dimensional tensors of shape ``[batch, dimension]`` containing logits.

.. code-block:: python

    @property
    def output_types(self):
        return {
            "out": [[NeuralType(('B', 'D'), LogitsType())]],
        }

Core APIs
=========

Base class for all NeMo models
------------------------------

.. autoclass:: nemo.core.ModelPT
    :show-inheritance:
    :members:
    :member-order: bysource
    :undoc-members: cfg, num_weights
    :exclude-members: set_eff_save, use_eff_save, teardown

Base Neural Module class
------------------------

.. autoclass:: nemo.core.NeuralModule
    :show-inheritance:
    :members:
    :member-order: bysource

Neural Type classes
-------------------

.. autoclass:: nemo.core.neural_types.NeuralType
    :show-inheritance:
    :members:
    :member-order: bysource

.. autoclass:: nemo.core.neural_types.axes.AxisType
    :show-inheritance:
    :members:
    :member-order: bysource

.. autoclass:: nemo.core.neural_types.elements.ElementType
    :show-inheritance:
    :members:
    :member-order: bysource

.. autoclass:: nemo.core.neural_types.comparison.NeuralTypeComparisonResult
    :show-inheritance:
    :members:
    :member-order: bysource

Experiment manager
------------------

.. autoclass:: nemo.utils.exp_manager.exp_manager
    :show-inheritance:
    :members:
    :member-order: bysource

