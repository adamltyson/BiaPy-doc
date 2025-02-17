.. _detection:

Detection
---------

The goal of this workflow is to localize objects in the input image, not requiring a pixel-level class. Common strategies produce either bounding boxes containing the objects or individual points at their center of mass :cite:p:`zhou2019objects`, which is the one adopted by BiaPy. 

* **Input:** 

  * Image (single-channel or multi-channel). E.g. image with shape ``(500, 500, 1)`` ``(y, x, channels)`` in ``2D`` or ``(100, 500, 500, 1)`` ``(z, y, x, channels)`` in ``3D``.  
  * ``.csv`` file containing the list of points to detect. 

* **Output:**

  * Image with the detected points as white dots.
  * ``.csv`` file with the list of detected points in napari format.
  * A ``_prob.csv`` file with the same list of points as above but now with their detection probability (also in napary format). 


In the figure below an example of this workflow's **input** is depicted:

.. list-table::
  :align: center
  :width: 680px

  * - .. figure:: ../img/detection_image_input.png
         :align: center
         :width: 300px

         Input image.  

    - .. figure:: ../img/detection_csv_input.svg
         :align: center
         :width: 300px

         Input ``.csv`` file. 

Description of the ``.csv`` file:
  
* Each row represents the middle point of the object to be detected. Each column is a coordinate in the image dimension space. 

* The first column name does not matter but it needs to be there. No matter also the enumeration and order for that column.

* If the images are ``3D``, three columns need to be present and their names must be ``[axis-0, axis-1, axis-2]``, which represent ``(z,y,x)`` axes. If the images are ``2D``, only two columns are required ``[axis-0, axis-1]``, which represent ``(y,x)`` axes. 

* For multi-class detection problem, i.e. ``MODEL.N_CLASSES > 1``, add an additional ``class`` column to the file. The classes need to start from ``1`` and consecutive, i.e. ``1,2,3,4...`` and not like ``1,4,8,6...``. 

* Coordinates can be float or int but they will be converted into ints so they can be translated to pixels. 

.. _detection_data_prep:

Data preparation
~~~~~~~~~~~~~~~~

To ensure the proper operation of the library the data directory tree should be something like this: 

.. collapse:: Expand directory tree 

    .. code-block:: bash
        
      dataset/
      ├── train
      │   ├── x
      │   │   ├── training-0001.tif
      │   │   ├── training-0002.tif
      │   │   ├── . . .
      │   │   ├── training-9999.tif
      │   └── y
      │       ├── training-0001.csv
      │       ├── training-0002.csv
      │       ├── . . .
      │       ├── training-9999.csv
      └── test
          ├── x
          │   ├── testing-0001.tif
          │   ├── testing-0002.tif
          │   ├── . . .
          │   ├── testing-9999.tif
          └── y
              ├── testing-0001.csv
              ├── testing-0002.csv
              ├── . . .
              ├── testing-9999.csv

\

.. warning:: In this workflow the name of each ``.tif`` file and its corresponding ``.csv`` file must be the same. 

Problem resolution
~~~~~~~~~~~~~~~~~~

Firstly, a **pre-processing** step is done where the list of points of the ``.csv`` file is transformed into point mask images. During this process some checks are made to ensure there is not repeated point in the ``.csv``. This option is ``True`` by default with ``PROBLEM.DETECTION.CHECK_POINTS_CREATED`` so if any problem is found the point mask of that ``.csv`` will not be created until the problem is solve. 

After the train phase, the model output will be an image where each pixel of each channel will have the probability (in ``[0-1]`` range) of being of the class that represents that channel. The image would be something similar to the left picture below:

.. list-table::
  :align: center
  :width: 680px

  * - .. figure:: ../img/detection_probs.png
         :align: center
         :width: 300px

         Model output.   

    - .. figure:: ../img/detected_points.png
         :align: center
         :width: 300px

         Final points considered. 


So those probability images, as the left picture above, can be converted into the final points, as the rigth picture above. To do so you can use two possible functions (defined by ``TEST.DET_POINT_CREATION_FUNCTION``):

* ``'peak_local_max'`` (`function <https://scikit-image.org/docs/stable/api/skimage.feature.html#skimage.feature.peak_local_max>`__). 
* ``'blob_log'`` (`function <https://scikit-image.org/docs/stable/api/skimage.feature.html#skimage.feature.blob_log>`__).  

The most important aspect of these options is using the threshold defined by the ``TEST.DET_MIN_TH_TO_BE_PEAK`` variable, which sets the minimum probability for a point to be considered.

Configuration file
~~~~~~~~~~~~~~~~~~

Find in `templates/detection <https://github.com/BiaPyX/BiaPy/tree/master/templates/detection>`__ folder of BiaPy a few YAML configuration templates for this workflow. 


Special workflow configuration
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Metrics
*******

During the inference phase the performance of the test data is measured using different metrics if test masks were provided (i.e. ground truth) and, consequently, ``DATA.TEST.LOAD_GT`` is ``True``. In the case of detection, the **Intersection over Union** (IoU) is measured after network prediction:

* **IoU** metric, also referred as the Jaccard index, is essentially a method to quantify the percent of overlap between the target mask and the prediction output. Depending on the configuration different values are calculated (as explained in :ref:`config_test` and :ref:`config_metric`). This values can vary a lot as stated in :cite:p:`Franco-Barranco2021`.

    * **Per patch**: IoU is calculated for each patch separately and then averaged. 
    * **Reconstructed image**: IoU is calculated for each reconstructed image separately and then averaged. Notice that depending on the amount of overlap/padding selected the merged image can be different than just concatenating each patch. 
    * **Full image**: IoU is calculated for each image separately and then averaged. The results may be slightly different from the reconstructed image. 

Then, after extracting the final points from the predictions, **precision**, **recall** and **F1** are defined as follows:

* **Precision**, is the fraction of relevant points among the retrieved points. More info `here <https://en.wikipedia.org/wiki/Precision_and_recall>`__.

* **Recall**, is the fraction of relevant points that were retrieved. More info `here <https://en.wikipedia.org/wiki/Precision_and_recall>`__.

* **F1**, is the harmonic mean of the precision and recall. More info `here <https://en.wikipedia.org/wiki/F-score>`__.

The last three metrics, i.e. precision, recall and F1, use ``TEST.DET_TOLERANCE`` to determine when a point is considered as a true positive. In this process the test resolution is also taken into account. You can set different tolerances for each class, e.g. ``[10,15]``.

Post-processing
***************

After network prediction and applied to ``3D`` images (e.g. ``PROBLEM.NDIM`` is ``2D`` or ``TEST.ANALIZE_2D_IMGS_AS_3D_STACK`` is ``True``). There are the following options:

* **Z-filtering**: to apply a median filtering in ``z`` axis. Useful to maintain class coherence across ``3D`` volumes. Enable it with ``TEST.POST_PROCESSING.Z_FILTERING`` and use ``TEST.POST_PROCESSING.Z_FILTERING_SIZE`` for the size of the median filter. 

* **YZ-filtering**: to apply a median filtering in ``y`` and ``z`` axes. Useful to maintain class coherence across ``3D`` volumes that can work slightly better than ``Z-filtering``. Enable it with ``TEST.POST_PROCESSING.YZ_FILTERING`` and use ``TEST.POST_PROCESSING.YZ_FILTERING_SIZE`` for the size of the median filter.  

\

Then, after extracting the final points from the predictions, the following post-processing methods are avaialable:
    
* **Remove close points**: to remove redundant close points to each other within a certain radius (controlled by ``TEST.POST_PROCESSING.REMOVE_CLOSE_POINTS``). The radius value can be specified using the variable ``TEST.POST_PROCESSING.REMOVE_CLOSE_POINTS_RADIUS``. You can set different radius for each class, e.g. ``[0.7,0.9]``. In this post-processing is important to set ``DATA.TEST.RESOLUTION``, specially for ``3D`` data where the resolution in ``z`` dimension is usually less than in other axes. That resolution will be taken into account when removing points. 
* **Create instances from points**: Once the points have been detected and any close points have been removed, it is possible to create instances from the remaining points. The variable ``TEST.POST_PROCESSING.DET_WATERSHED`` can be set to perform this step. However, sometimes cells have low contrast in their centers, for example due to the presence of a nucleus. This can result in the seed growing to fill only the nucleus while the cell is much larger. In order to address the issue of limited growth of certain types of seeds, a process has been implemented to expand the seeds beyond the borders of their nuclei. This process allows for improved growth of these seeds. To ensure that this process is applied only to the appropriate cells, variables such as ``TEST.POST_PROCESSING.DET_WATERSHED_DONUTS_CLASSES``, ``TEST.POST_PROCESSING.DET_WATERSHED_DONUTS_PATCH``, and ``TEST.POST_PROCESSING.DET_WATERSHED_DONUTS_NUCLEUS_DIAMETER`` have been created. It is important to note that these variables are necessary to prevent the expansion of the seed beyond the boundaries of the cell, which could lead to expansion into the background.

.. figure:: ../img/donuts_cell_det_watershed_illustration.png
    :width: 400px
    :align: center
    
    For left to right: raw image, initial seeds for the watershed and the resulting instances after growing the seeds. In the first row the problem with nucleus visible type cells is depicted, where the central seed can not be grown more than the nucleus border. On the second row the solution of dilating the central point is depicted. 

Run
~~~

.. tabs::
   .. tab:: GUI

        Select detection workflow during the creation of a new configuration file:

        .. image:: https://raw.githubusercontent.com/BiaPyX/BiaPy-doc/master/source/img/gui/biapy_gui_detection.jpg
            :align: center 

   .. tab:: Google Colab
        Two different options depending on the image dimension: 

        .. |detection_2D_colablink| image:: https://colab.research.google.com/assets/colab-badge.svg
            :target: https://colab.research.google.com/github/BiaPyX/BiaPy/blob/master/notebooks/detection/BiaPy_2D_Detection.ipynb

        * 2D: |detection_2D_colablink|

        .. |detection_3D_colablink| image:: https://colab.research.google.com/assets/colab-badge.svg
            :target: https://colab.research.google.com/github/BiaPyX/BiaPy/blob/master/notebooks/detection/BiaPy_3D_Detection.ipynb

        * 3D: |detection_3D_colablink|

   .. tab:: Docker 

        `Open a terminal <../get_started/faq.html#opening-a-terminal>`__ as described in :ref:`installation`. For instance, using `2d_detection.yaml <https://github.com/BiaPyX/BiaPy/blob/master/templates/detection/2d_detection.yaml>`__ template file, the code can be run as follows:

        .. code-block:: bash                                                                                                    

            # Configuration file
            job_cfg_file=/home/user/2d_detection.yaml
            # Path to the data directory
            data_dir=/home/user/data
            # Where the experiment output directory should be created
            result_dir=/home/user/exp_results
            # Just a name for the job
            job_name=my_2d_detection
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
            Note that ``data_dir`` must contain all the paths ``DATA.*.PATH`` and ``DATA.*.GT_PATH`` so the container can find them. For instance, if you want to only train in this example ``DATA.TRAIN.PATH`` and ``DATA.TRAIN.GT_PATH`` could be ``/home/user/data/train/x`` and ``/home/user/data/train/y`` respectively. 

   .. tab:: Command line

        `Open a terminal <../get_started/faq.html#opening-a-terminal>`__ as described in :ref:`installation`. For instance, using `2d_detection.yaml <https://github.com/BiaPyX/BiaPy/blob/master/templates/detection/2d_detection.yaml>`__ template file, the code can be run as follows:

        .. code-block:: bash
            
            # Configuration file
            job_cfg_file=/home/user/2d_detection.yaml       
            # Where the experiment output directory should be created
            result_dir=/home/user/exp_results  
            # Just a name for the job
            job_name=my_2d_detection      
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

   
.. _detection_results:

Results                                                                                                                 
~~~~~~~  

The results are placed in ``results`` folder under ``--result_dir`` directory with the ``--name`` given. Following the example, you should see that the directory ``/home/user/exp_results/my_2d_detection`` has been created. If the same experiment is run 5 times, varying ``--run_id`` argument only, you should find the following directory tree: 

.. collapse:: Expand directory tree 

    .. code-block:: bash

      my_2d_detection/
      ├── config_files
      │   └── my_2d_detection.yaml                                                                                                           
      ├── checkpoints
      │   └── my_2d_detection_1-checkpoint-best.pth
      └── results
          ├── my_2d_detection_1
          ├── . . .
          └── my_2d_detection_5
              ├── aug
              │   └── .tif files
              ├── charts
              │   ├── my_2d_detection_1_*.png
              │   └── my_2d_detection_1_loss.png
              ├── per_image
              │   ├── .tif files
              │   └── .zarr files (or.h5)
              ├── full_image
              │   └── .tif files
              ├── per_image_local_max_check
              │   ├── .tif files  
              │   ├── *_points.csv files  
              │   └── *_all_points.csv files
              ├── point_associations
              │   ├── .tif files
              │   └── .csv files  
              ├── watershed
              │   ├── seed_map.tif
              │   ├── foreground.tif                
              │   └── watershed.tif
              ├── train_logs
              └── tensorboard

\

* ``config_files``: directory where the .yaml filed used in the experiment is stored. 

  * ``my_2d_detection.yaml``: YAML configuration file used (it will be overwrited every time the code is run).

* ``checkpoints``, *optional*: directory where model's weights are stored. Only created when ``TRAIN.ENABLE`` is ``True`` and the model is trained for at least one epoch. Can contain:

  * ``my_2d_detection_1-checkpoint-best.pth``, *optional*: checkpoint file (best in validation) where the model's weights are stored among other information. Only created when the model is trained for at least one epoch. 

  * ``normalization_mean_value.npy``, *optional*: normalization mean value. Is saved to not calculate it everytime and to use it in inference. Only created if ``DATA.NORMALIZATION.TYPE`` is ``custom``.
  
  * ``normalization_std_value.npy``, *optional*: normalization std value. Is saved to not calculate it everytime and to use it in inference. Only created if ``DATA.NORMALIZATION.TYPE`` is ``custom``.
  
* ``results``: directory where all the generated checks and results will be stored. There, one folder per each run are going to be placed. Can contain:

  * ``my_2d_detection_1``: run 1 experiment folder. Can contain:

    * ``aug``, *optional*: image augmentation samples. Only created if ``AUGMENTOR.AUG_SAMPLES`` is ``True``.

    * ``charts``, *optional*: only created when ``TRAIN.ENABLE`` is ``True`` and epochs trained are more or equal ``LOG.CHART_CREATION_FREQ``. Can contain:

      * ``my_2d_detection_1_*.png``: plot of each metric used during training.

      * ``my_2d_detection_1_loss.png``: loss over epochs plot. 

    * ``per_image``, *optional*: only created if ``TEST.FULL_IMG`` is ``False``. Can contain:

      * ``.tif files``, *optional*: reconstructed images from patches. Created when ``TEST.BY_CHUNKS.ENABLE`` is ``False`` or when ``TEST.BY_CHUNKS.ENABLE`` is ``True`` but ``TEST.BY_CHUNKS.SAVE_OUT_TIF`` is ``True``. 

      * ``.zarr files (or.h5)``, *optional*: reconstructed images from patches. Created when ``TEST.BY_CHUNKS.ENABLE`` is ``True``.

    * ``full_image``, *optional*: only created if ``TEST.FULL_IMG`` is ``True``. Can contain:

      * ``.tif files``: full image predictions.

    * ``per_image_local_max_check``, can contain:

      * ``.tif files``, *optional*: same as ``per_image`` but with the final detected points in tif format. Created when ``TEST.BY_CHUNKS.ENABLE`` is ``False`` or when ``TEST.BY_CHUNKS.ENABLE`` is ``True`` but ``TEST.BY_CHUNKS.SAVE_OUT_TIF`` is ``True``.  

      * ``*_points.csv files``: final point list for each test sample or test chunk (only created if ``TEST.BY_CHUNKS.ENABLE`` is ``True``). 

      * ``*_all_points.csv files``, *optional*: all points of all chunks together for each test Zarr/H5 sample (only created if ``TEST.BY_CHUNKS.ENABLE``).

    * ``point_associations``, *optional*: only if ground truth was provided by setting ``DATA.TEST.LOAD_GT``. Can contain:

      * ``.tif files``, coloured associations per each matching threshold selected to be analised (controlled by ``TEST.MATCHING_STATS_THS_COLORED_IMG``) for each test sample or test chunk. Green is a true positive, red is a false negative and blue is a false positive.

      * ``.csv files``: false positives (``_fp``) and ground truth associations (``_gt_assoc``) for each test sample or test chunk. There is a file per each matching threshold selected (controlled by ``TEST.MATCHING_STATS_THS``).  

    * ``watershed``, *optional*: only if ``TEST.POST_PROCESSING.DET_WATERSHED`` and ``PROBLEM.DETECTION.DATA_CHECK_MW`` are ``True``. Can contain: 
              
      * ``seed_map.tif``: initial seeds created before growing. 
          
      * ``semantic.tif``: region where the watershed will run.

      * ``foreground.tif``: foreground mask area that delimits the grown of the seeds.

    * ``train_logs``: each row represents a summary of each epoch stats. Only avaialable if training was done.
        
    * ``tensorboard``: tensorboard logs.

.. note:: 

  Here, for visualization purposes, only ``my_2d_detection_1`` has been described but ``my_2d_detection_2``, ``my_2d_detection_3``, ``my_2d_detection_4`` and ``my_2d_detection_5`` will follow the same structure.



