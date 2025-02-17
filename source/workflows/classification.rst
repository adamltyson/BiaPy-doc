.. _classification:

Classification
--------------

The goal of this workflow is to assign a label to the input image. 

* **Input:** 

  * Image (single-channel or multi-channel). E.g. image with shape ``(500, 500, 1)`` ``(y, x, channels)`` in ``2D`` or ``(100, 500, 500, 1)`` ``(z, y, x, channels)`` in ``3D``. 

* **Output:**

  * ``.csv`` file with the assigned class to each image.

In the figure below a few examples of this workflow's **input** are depicted:

.. list-table::
  :align: center
  :width: 680px
  
  * - .. figure:: ../img/classification/MedMNIST_DermaMNIST_test1008_0.png
         :align: center
         :width: 50

    - .. figure:: ../img/classification/MedMNIST_DermaMNIST_test10_1.png
         :align: center
         :width: 50
         
    - .. figure:: ../img/classification/MedMNIST_DermaMNIST_test1002_2.png
         :align: center
         :width: 50

    - .. figure:: ../img/classification/MedMNIST_DermaMNIST_test1030_3.png
         :align: center
         :width: 50

    - .. figure:: ../img/classification/MedMNIST_DermaMNIST_test1003_4.png
         :align: center
         :width: 50

    - .. figure:: ../img/classification/MedMNIST_DermaMNIST_test0_5.png
         :align: center
         :width: 50

    - .. figure:: ../img/classification/MedMNIST_DermaMNIST_test1021_6.png
         :align: center
         :width: 50

Each of these examples are of a different class and were obtained from `MedMNIST v2 <https://medmnist.com/>`__ (:cite:p:`yang2021medmnist`), concretely from DermaMNIST dataset which is a large collection of multi-source dermatoscopic images of common
pigmented skin lesions.


.. _classification_data_prep:

Data preparation
~~~~~~~~~~~~~~~~

Each image label is obtained from the directory name in which that image resides. That is why is so important to follow the directory tree as described below. If you have a .csv file with each image label, as is provided by `MedMNIST v2 <https://medmnist.com/>`__, you can use our script `from_class_csv_to_folders.py <https://github.com/BiaPyX/BiaPy/blob/master/biapy/utils/scripts/from_class_csv_to_folders.py>`__ to create the directory tree as below: 
    
.. collapse:: Expand directory tree 

    .. code-block:: bash
        
      dataset/
      ├── train
      │   ├── 0
      │   │   ├── train0_0.png
      │   │   ├── train1013_0.png
      │   │   ├── . . .
      │   │   └── train932_0.png
      │   ├── 1
      │   │   ├── train104_1.png
      │   │   ├── train1049_1.png
      │   │   ├── . . .
      │   │   └── train964_1.png
      | . . .
      │   └── 6
      │       ├── train1105_6.png
      │       ├── train1148_6.png
      │       ├── . . .
      │       └── train98_6.png
      └── test
          ├── 0
          │   ├── test1008_0.png
          │   ├── test1084_0.png
          │   ├── . . .
          │   └── test914_0.png
          ├── 1
          │   ├── test10_1.png
          │   ├── test1034_1.png
          │   ├── . . .
          │   └── test984_1.png
        . . .
          └── 6
              ├── test1021_6.png
              ├── test1069_6.png
              ├── . . .
              └── test806_6.png

\

Here each directory is a number but it can be any string. Notice that they will be considered the class names. Regarding the test, if you have no classes it doesn't matter if the images are separated in several folders or are all in one folder. But, if ``DATA.TEST.LOAD_GT`` is ``True``, each folder in test path (i.e. ``DATA.TEST.PATH``) will be considered as a class (as done for training and validation). 

.. _classification_problem_resolution:

Configuration file
~~~~~~~~~~~~~~~~~~

Find in `templates/classification <https://github.com/BiaPyX/BiaPy/tree/master/templates/classification>`__ folder of BiaPy a few YAML configuration templates for this workflow. 


Special workflow configuration
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Metrics
*******

During the inference phase the performance of the test data is measured using different metrics if test masks were provided (i.e. ground truth) and, consequently, ``DATA.TEST.LOAD_GT`` is ``True``. In the case of classification the **accuracy**, **precision**, **recall**, and **F1** are calculated. Apart from that, the `confusion matrix <https://en.wikipedia.org/wiki/Confusion_matrix>`__ is also printed.

Run
~~~

.. tabs::
   .. tab:: GUI

        Select classification workflow during the creation of a new configuration file:

        .. image:: https://raw.githubusercontent.com/BiaPyX/BiaPy-doc/master/source/img/gui/biapy_gui_classification.jpg
            :align: center 

   .. tab:: Google Colab 

        Two different options depending on the image dimension:

        .. |class_2D_colablink| image:: https://colab.research.google.com/assets/colab-badge.svg
            :target: https://colab.research.google.com/github/BiaPyX/BiaPy/blob/master/notebooks/classification/BiaPy_2D_Classification.ipynb

        * 2D: |class_2D_colablink|

        .. |class_3D_colablink| image:: https://colab.research.google.com/assets/colab-badge.svg
            :target: https://colab.research.google.com/github/BiaPyX/BiaPy/blob/master/notebooks/classification/BiaPy_3D_Classification.ipynb

        * 3D: |class_3D_colablink|

   .. tab:: Docker

        `Open a terminal <../get_started/faq.html#opening-a-terminal>`__ as described in :ref:`installation`. For instance, using `2d_classification.yaml <https://github.com/BiaPyX/BiaPy/blob/master/templates/classification/2d_classification.yaml>`__ template file, the code can be run as follows:

        .. code-block:: bash                                                                                                    

            # Configuration file
            job_cfg_file=/home/user/2d_classification.yaml
            # Path to the data directory
            data_dir=/home/user/data
            # Where the experiment output directory should be created
            result_dir=/home/user/exp_results
            # Just a name for the job
            job_name=classification
            # Number that should be increased when one need to run the same job multiple times (reproducibility)
            job_counter=1
            # Number of the GPU to run the job in (according to 'nvidia-smi' command)
            gpu_number=0

            docker run --rm \
                --gpus "device=$gpu_number" \
                --mount type=bind,source=$job_cfg_file,target=$job_cfg_file \
                --mount type=bind,source=$result_dir,target=$result_dir \
                --mount type=bind,source=$data_dir,target=$data_dir \
                BiaPyX/biapy \
                    -cfg $job_cfg_file \
                    -rdir $result_dir \
                    -name $job_name \
                    -rid $job_counter \
                    -gpu "$gpu_number"

        .. note:: 
            Note that ``data_dir`` must contain the path ``DATA.*.PATH`` so the container can find it. For instance, if you want to only train in this example ``DATA.TRAIN.PATH`` could be ``/home/user/data/train/``. 

   .. tab:: Command line 

        `Open a terminal <../get_started/faq.html#opening-a-terminal>`__ as described in :ref:`installation`. For instance, using `2d_classification.yaml <https://github.com/BiaPyX/BiaPy/blob/master/templates/classification/2d_classification.yaml>`__ template file, the code can be run as follows:

        .. code-block:: bash
            
            # Configuration file
            job_cfg_file=/home/user/2d_classification.yaml       
            # Where the experiment output directory should be created
            result_dir=/home/user/exp_results  
            # Just a name for the job
            job_name=my_2d_classification      
            # Number that should be increased when one need to run the same job multiple times (reproducibility)
            job_counter=1
            # Number of the GPU to run the job in (according to 'nvidia-smi' command)
            gpu_number=0                   

            # Load the environment
            conda activate BiaPy_env
            
            biapy \
                --config $job_cfg_file \
                --result_dir $result_dir  \ 
                --name $job_name    \
                --run_id $job_counter  \
                --gpu "$gpu_number"  

        For multi-GPU training you can call BiaPy as follows:

        .. code-block:: bash
            
            # First check where is your biapy command (you need it in the below command)
            # $ which biapy
            # > /home/user/anaconda3/envs/BiaPy_env/bin/biapy

            gpu_number="0, 1, 2"
            python -u -m torch.distributed.run \
                --nproc_per_node=3 \
                /home/user/anaconda3/envs/BiaPy_env/bin/biapy \
                --config $job_cfg_file \
                --result_dir $result_dir  \ 
                --name $job_name    \
                --run_id $job_counter  \
                --gpu "$gpu_number"  

        ``nproc_per_node`` needs to be equal to the number of GPUs you are using (e.g. ``gpu_number`` length).

.. _classification_results:

Results                                                                                                                 
~~~~~~~  

The main output of this workflow will be a file named ``predictions.csv`` that will contain the predicted image class:

.. figure:: ../img/classification/classification_csv_output.svg
    :align: center
    :width: 150

    Classification workflow output

All files are placed in ``results`` folder under ``--result_dir`` directory with the ``--name`` given. Following the example, you should see that the directory ``/home/user/exp_results/classification`` has been created. If the same experiment is run 5 times, varying ``--run_id`` argument only, you should find the following directory tree: 

.. collapse:: Expand directory tree 

    .. code-block:: bash
        
      my_2d_classification/
      ├── config_files
      │   └── 2d_classification.yaml                                                                                                           
      ├── checkpoints
      │   └── model_weights_classification_1.h5
      └── results
          ├── my_2d_classification_1
          ├── . . .
          └── my_2d_classification_5
              ├── predictions.csv
              ├── aug
              │   └── .tif files
              ├── charts
              │   ├── my_2d_classification_1_*.png
              │   └── my_2d_classification_1_loss.png
              ├── train_logs
              └── tensorboard

\

* ``config_files``: directory where the .yaml filed used in the experiment is stored. 

  * ``2d_classification.yaml``: YAML configuration file used (it will be overwrited every time the code is run).

* ``checkpoints``, *optional*: directory where model's weights are stored. Only created when ``TRAIN.ENABLE`` is ``True`` and the model is trained for at least one epoch. 

  * ``model_weights_my_2d_classification_1.h5``, *optional*: checkpoint file (best in validation) where the model's weights are stored among other information. Only created when the model is trained for at least one epoch. 
  
  * ``normalization_mean_value.npy``, *optional*: normalization mean value. Is saved to not calculate it everytime and to use it in inference. Only created if ``DATA.NORMALIZATION.TYPE`` is ``custom``.
  
  * ``normalization_std_value.npy``, *optional*: normalization std value. Is saved to not calculate it everytime and to use it in inference. Only created if ``DATA.NORMALIZATION.TYPE`` is ``custom``.

* ``results``: directory where all the generated checks and results will be stored. There, one folder per each run are going to be placed.

  * ``my_2d_classification_1``: run 1 experiment folder. Can contain:

    * ``predictions.csv``: list of assigned class per test image.

    * ``aug``, *optional*: image augmentation samples. Only created if ``AUGMENTOR.AUG_SAMPLES`` is ``True``.

    * ``charts``, *optional*. Only created when ``TRAIN.ENABLE`` is ``True`` and epochs trained are more or equal ``LOG.CHART_CREATION_FREQ``:  

      * ``my_2d_classification_1_*.png``: plot of each metric used during training. 

      * ``my_2d_classification_1_loss.png``: loss over epochs plot. 

    * ``train_logs``: each row represents a summary of each epoch stats. Only avaialable if training was done.

    * ``tensorboard``: tensorboard logs.

.. note:: 

  Here, for visualization purposes, only ``my_2d_classification_1`` has been described but ``my_2d_classification_2``, ``my_2d_classification_3``, ``my_2d_classification_4`` and ``my_2d_classification_5`` directories will follow the same structure.



