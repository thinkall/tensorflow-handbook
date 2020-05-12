TensorFlow常用模塊
=====================================

.. admonition:: 前置知識

    * `Python的序列化模塊Pickle <http://www.runoob.com/python3/python3-inputoutput.html>`_ （非必須）
    * `Python的特殊函數參數**kwargs <https://eastlakeside.gitbooks.io/interpy-zh/content/args_kwargs/Usage_kwargs.html>`_ （非必須）
    * `Python的疊代器 <https://www.runoob.com/python3/python3-iterator-generator.html>`_ 

``tf.train.Checkpoint`` ：變量的保存與恢復
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

..
    https://www.tensorflow.org/beta/guide/checkpoints

.. warning:: Checkpoint只保存模型的參數，不保存模型的計算過程，因此一般用於在具有模型原始碼的時候恢復之前訓練好的模型參數。如果需要導出模型（無需原始碼也能運行模型），請參考 :ref:`「部署」章節中的SavedModel <savedmodel>` 。

很多時候，我們希望在模型訓練完成後能將訓練好的參數（變量）保存起來。在需要使用模型的其他地方載入模型和參數，就能直接得到訓練好的模型。可能你第一個想到的是用Python的序列化模塊 ``pickle`` 存儲 ``model.variables``。但不幸的是，TensorFlow的變量類型 ``ResourceVariable`` 並不能被序列化。

好在TensorFlow提供了 ``tf.train.Checkpoint`` 這一強大的變量保存與恢復類，可以使用其 ``save()`` 和 ``restore()`` 方法將TensorFlow中所有包含Checkpointable State的對象進行保存和恢復。具體而言，``tf.keras.optimizer`` 、 ``tf.Variable`` 、 ``tf.keras.Layer`` 或者 ``tf.keras.Model`` 實例都可以被保存。其使用方法非常簡單，我們首先聲明一個Checkpoint：

.. code-block:: python

    checkpoint = tf.train.Checkpoint(model=model)

這裡 ``tf.train.Checkpoint()`` 接受的初始化參數比較特殊，是一個 ``**kwargs`` 。具體而言，是一系列的鍵值對，鍵名可以隨意取，值爲需要保存的對象。例如，如果我們希望保存一個繼承 ``tf.keras.Model`` 的模型實例 ``model`` 和一個繼承 ``tf.train.Optimizer`` 的優化器 ``optimizer`` ，我們可以這樣寫：

.. code-block:: python

    checkpoint = tf.train.Checkpoint(myAwesomeModel=model, myAwesomeOptimizer=optimizer)

這裡 ``myAwesomeModel`` 是我們爲待保存的模型 ``model`` 所取的任意鍵名。注意，在恢復變量的時候，我們還將使用這一鍵名。

接下來，當模型訓練完成需要保存的時候，使用：

.. code-block:: python

    checkpoint.save(save_path_with_prefix)

就可以。 ``save_path_with_prefix`` 是保存文件的目錄+前綴。

.. note:: 例如，在原始碼目錄建立一個名爲save的文件夾並調用一次 ``checkpoint.save('./save/model.ckpt')`` ，我們就可以在可以在save目錄下發現名爲 ``checkpoint`` 、  ``model.ckpt-1.index`` 、 ``model.ckpt-1.data-00000-of-00001`` 的三個文件，這些文件就記錄了變量信息。``checkpoint.save()`` 方法可以運行多次，每運行一次都會得到一個.index文件和.data文件，序號依次累加。

當在其他地方需要爲模型重新載入之前保存的參數時，需要再次實例化一個checkpoint，同時保持鍵名的一致。再調用checkpoint的restore方法。就像下面這樣：

.. code-block:: python

    model_to_be_restored = MyModel()                                        # 待恢復參數的同一模型
    checkpoint = tf.train.Checkpoint(myAwesomeModel=model_to_be_restored)   # 鍵名保持爲「myAwesomeModel」
    checkpoint.restore(save_path_with_prefix_and_index)

即可恢復模型變量。 ``save_path_with_prefix_and_index`` 是之前保存的文件的目錄+前綴+編號。例如，調用 ``checkpoint.restore('./save/model.ckpt-1')`` 就可以載入前綴爲 ``model.ckpt`` ，序號爲1的文件來恢復模型。

當保存了多個文件時，我們往往想載入最近的一個。可以使用 ``tf.train.latest_checkpoint(save_path)`` 這個輔助函數返回目錄下最近一次checkpoint的文件名。例如如果save目錄下有 ``model.ckpt-1.index`` 到 ``model.ckpt-10.index`` 的10個保存文件， ``tf.train.latest_checkpoint('./save')`` 即返回 ``./save/model.ckpt-10`` 。

總體而言，恢復與保存變量的典型代碼框架如下：

.. code-block:: python

    # train.py 模型訓練階段

    model = MyModel()
    # 實例化Checkpoint，指定保存對象爲model（如果需要保存Optimizer的參數也可加入）
    checkpoint = tf.train.Checkpoint(myModel=model)     
    # ...（模型訓練代碼）
    # 模型訓練完畢後將參數保存到文件（也可以在模型訓練過程中每隔一段時間就保存一次）
    checkpoint.save('./save/model.ckpt')               

.. code-block:: python

    # test.py 模型使用階段

    model = MyModel()
    checkpoint = tf.train.Checkpoint(myModel=model)             # 實例化Checkpoint，指定恢復對象爲model
    checkpoint.restore(tf.train.latest_checkpoint('./save'))    # 從文件恢復模型參數
    # 模型使用代碼

.. note:: ``tf.train.Checkpoint`` 與以前版本常用的 ``tf.train.Saver`` 相比，強大之處在於其支持在即時執行模式下「延遲」恢復變量。具體而言，當調用了 ``checkpoint.restore()`` ，但模型中的變量還沒有被建立的時候，Checkpoint可以等到變量被建立的時候再進行數值的恢復。即時執行模式下，模型中各個層的初始化和變量的建立是在模型第一次被調用的時候才進行的（好處在於可以根據輸入的張量形狀而自動確定變量形狀，無需手動指定）。這意味著當模型剛剛被實例化的時候，其實裡面還一個變量都沒有，這時候使用以往的方式去恢復變量數值是一定會報錯的。比如，你可以試試在train.py調用 ``tf.keras.Model`` 的 ``save_weight()`` 方法保存model的參數，並在test.py中實例化model後立即調用 ``load_weight()`` 方法，就會出錯，只有當調用了一遍model之後再運行 ``load_weight()`` 方法才能得到正確的結果。可見， ``tf.train.Checkpoint`` 在這種情況下可以給我們帶來相當大的便利。另外， ``tf.train.Checkpoint`` 同時也支持圖執行模式。

最後提供一個實例，以前章的 :ref:`多層感知機模型 <mlp>` 爲例展示模型變量的保存和載入：

.. literalinclude:: /_static/code/zh/tools/save_and_restore/mnist.py
    :emphasize-lines: 20, 30-32, 38-39

在代碼目錄下建立save文件夾並運行代碼進行訓練後，save文件夾內將會存放每隔100個batch保存一次的模型變量數據。在命令行參數中加入 ``--mode=test`` 並再次運行代碼，將直接使用最後一次保存的變量值恢復模型並在測試集上測試模型性能，可以直接獲得95%左右的準確率。

.. admonition:: 使用 ``tf.train.CheckpointManager`` 刪除舊的Checkpoint以及自定義文件編號

    在模型的訓練過程中，我們往往每隔一定步數保存一個Checkpoint並進行編號。不過很多時候我們會有這樣的需求：

    - 在長時間的訓練後，程序會保存大量的Checkpoint，但我們只想保留最後的幾個Checkpoint；
    - Checkpoint默認從1開始編號，每次累加1，但我們可能希望使用別的編號方式（例如使用當前Batch的編號作爲文件編號）。

    這時，我們可以使用TensorFlow的 ``tf.train.CheckpointManager`` 來實現以上需求。具體而言，在定義Checkpoint後接著定義一個CheckpointManager：

    .. code-block:: python

        checkpoint = tf.train.Checkpoint(model=model)
        manager = tf.train.CheckpointManager(checkpoint, directory='./save', checkpoint_name='model.ckpt', max_to_keep=k)

    此處， ``directory`` 參數爲文件保存的路徑， ``checkpoint_name`` 爲文件名前綴（不提供則默認爲 ``ckpt`` ）， ``max_to_keep`` 爲保留的Checkpoint數目。

    在需要保存模型的時候，我們直接使用 ``manager.save()`` 即可。如果我們希望自行指定保存的Checkpoint的編號，則可以在保存時加入 ``checkpoint_number`` 參數。例如 ``manager.save(checkpoint_number=100)`` 。

    以下提供一個實例，展示使用CheckpointManager限制僅保留最後三個Checkpoint文件，並使用batch的編號作爲Checkpoint的文件編號。

    .. literalinclude:: /_static/code/zh/tools/save_and_restore/mnist_manager.py
        :emphasize-lines: 22, 34

TensorBoard：訓練過程可視化
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

..
    https://www.tensorflow.org/tensorboard/r2/get_started

有時，你希望查看模型訓練過程中各個參數的變化情況（例如損失函數loss的值）。雖然可以通過命令行輸出來查看，但有時顯得不夠直觀。而TensorBoard就是一個能夠幫助我們將訓練過程可視化的工具。

實時查看參數變化情況
-------------------------------------------

首先在代碼目錄下建立一個文件夾（如 ``./tensorboard`` ）存放TensorBoard的記錄文件，並在代碼中實例化一個記錄器：

.. code-block:: python
    
    summary_writer = tf.summary.create_file_writer('./tensorboard')     # 參數爲記錄文件所保存的目錄

接下來，當需要記錄訓練過程中的參數時，通過with語句指定希望使用的記錄器，並對需要記錄的參數（一般是scalar）運行 ``tf.summary.scalar(name, tensor, step=batch_index)`` ，即可將訓練過程中參數在step時候的值記錄下來。這裡的step參數可根據自己的需要自行制定，一般可設置爲當前訓練過程中的batch序號。整體框架如下：

.. code-block:: python

    summary_writer = tf.summary.create_file_writer('./tensorboard')    
    # 開始模型訓練
    for batch_index in range(num_batches):
        # ...（訓練代碼，當前batch的損失值放入變量loss中）
        with summary_writer.as_default():                               # 希望使用的記錄器
            tf.summary.scalar("loss", loss, step=batch_index)
            tf.summary.scalar("MyScalar", my_scalar, step=batch_index)  # 還可以添加其他自定義的變量

每運行一次 ``tf.summary.scalar()`` ，記錄器就會向記錄文件中寫入一條記錄。除了最簡單的標量（scalar）以外，TensorBoard還可以對其他類型的數據（如圖像，音頻等）進行可視化，詳見 `TensorBoard文檔 <https://www.tensorflow.org/tensorboard/r2/get_started>`_ 。

當我們要對訓練過程可視化時，在代碼目錄打開終端（如需要的話進入TensorFlow的conda環境），運行::

    tensorboard --logdir=./tensorboard

然後使用瀏覽器訪問命令行程序所輸出的網址（一般是http://name-of-your-computer:6006），即可訪問TensorBoard的可視界面，如下圖所示：

.. figure:: /_static/image/tools/tensorboard.png
    :width: 100%
    :align: center

默認情況下，TensorBoard每30秒更新一次數據。不過也可以點擊右上角的刷新按鈕手動刷新。

TensorBoard的使用有以下注意事項：

* 如果需要重新訓練，需要刪除掉記錄文件夾內的信息並重啓TensorBoard（或者建立一個新的記錄文件夾並開啓TensorBoard， ``--logdir`` 參數設置爲新建立的文件夾）；
* 記錄文件夾目錄保持全英文。

.. _graph_profile:

查看Graph和Profile信息
-------------------------------------------

除此以外，我們可以在訓練時使用 ``tf.summary.trace_on`` 開啓Trace，此時TensorFlow會將訓練時的大量信息（如計算圖的結構，每個操作所耗費的時間等）記錄下來。在訓練完成後，使用 ``tf.summary.trace_export`` 將記錄結果輸出到文件。

.. code-block:: python

    tf.summary.trace_on(graph=True, profiler=True)  # 開啓Trace，可以記錄圖結構和profile信息
    # 進行訓練
    with summary_writer.as_default():
        tf.summary.trace_export(name="model_trace", step=0, profiler_outdir=log_dir)    # 保存Trace信息到文件

之後，我們就可以在TensorBoard中選擇「Profile」，以時間軸的方式查看各操作的耗時情況。如果使用了 :ref:`tf.function <tffunction>` 建立了計算圖，也可以點擊「Graphs」查看圖結構。

.. figure:: /_static/image/tools/profiling.png
    :width: 100%
    :align: center

.. figure:: /_static/image/tools/graph.png
    :width: 100%
    :align: center

實例：查看多層感知機模型的訓練情況
-------------------------------------------

最後提供一個實例，以前章的 :ref:`多層感知機模型 <mlp>` 爲例展示TensorBoard的使用：

.. literalinclude:: /_static/code/zh/tools/tensorboard/mnist.py
    :emphasize-lines: 12-13, 21-22, 25-26

.. _tfdata:

``tf.data`` ：數據集的構建與預處理
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

..
    https://www.tensorflow.org/beta/guide/data

很多時候，我們希望使用自己的數據集來訓練模型。然而，面對一堆格式不一的原始數據文件，將其預處理並讀入程序的過程往往十分繁瑣，甚至比模型的設計還要耗費精力。比如，爲了讀入一批圖像文件，我們可能需要糾結於python的各種圖像處理包（比如 ``pillow`` ），自己設計Batch的生成方式，最後還可能在運行的效率上不盡如人意。爲此，TensorFlow提供了 ``tf.data`` 這一模塊，包括了一套靈活的數據集構建API，能夠幫助我們快速、高效地構建數據輸入的流水線，尤其適用於數據量巨大的場景。

數據集對象的建立
-------------------------------------------

``tf.data`` 的核心是 ``tf.data.Dataset`` 類，提供了對數據集的高層封裝。``tf.data.Dataset`` 由一系列的可疊代訪問的元素（element）組成，每個元素包含一個或多個張量。比如說，對於一個由圖像組成的數據集，每個元素可以是一個形狀爲 ``長×寬×通道數`` 的圖片張量，也可以是由圖片張量和圖片標籤張量組成的元組（Tuple）。

最基礎的建立 ``tf.data.Dataset`` 的方法是使用 ``tf.data.Dataset.from_tensor_slices()`` ，適用於數據量較小（能夠整個裝進內存）的情況。具體而言，如果我們的數據集中的所有元素通過張量的第0維，拼接成一個大的張量（例如，前節的MNIST數據集的訓練集即爲一個 ``[60000, 28, 28, 1]`` 的張量，表示了60000張28*28的單通道灰度圖像），那麼我們提供一個這樣的張量或者第0維大小相同的多個張量作爲輸入，即可按張量的第0維展開來構建數據集，數據集的元素數量爲張量第0位的大小。具體示例如下：

.. literalinclude:: /_static/code/zh/tools/tfdata/tutorial.py
    :lines: 1-14
    :emphasize-lines: 11

輸出::

    2013 12000
    2014 14000
    2015 15000
    2016 16500
    2017 17500

.. warning:: 當提供多個張量作爲輸入時，張量的第0維大小必須相同，且必須將多個張量作爲元組（Tuple，即使用Python中的小括號）拼接並作爲輸入。

類似地，我們可以載入前章的MNIST數據集：

.. literalinclude:: /_static/code/zh/tools/tfdata/tutorial.py
    :lines: 16-25
    :emphasize-lines: 5

輸出

.. figure:: /_static/image/tools/mnist_1.png
    :width: 40%
    :align: center

.. hint:: TensorFlow Datasets提供了一個基於 ``tf.data.Datasets`` 的開箱即用的數據集集合，相關內容可參考 :doc:`TensorFlow Datasets <../appendix/tfds>` 。例如，使用以下語句：

    .. code-block:: python

        import tensorflow_datasets as tfds
        dataset = tfds.load("mnist", split=tfds.Split.TRAIN, as_supervised=True)

    即可快速載入MNIST數據集。

對於特別巨大而無法完整載入內存的數據集，我們可以先將數據集處理爲 TFRecord 格式，然後使用 ``tf.data.TFRocrdDataset()`` 進行載入。詳情請參考 :ref:`後節 <tfrecord>`：

數據集對象的預處理
-------------------------------------------

``tf.data.Dataset`` 類爲我們提供了多種數據集預處理方法。最常用的如：

- ``Dataset.map(f)`` ：對數據集中的每個元素應用函數 ``f`` ，得到一個新的數據集（這部分往往結合 ``tf.io`` 進行讀寫和解碼文件， ``tf.image`` 進行圖像處理）；
- ``Dataset.shuffle(buffer_size)`` ：將數據集打亂（設定一個固定大小的緩衝區（Buffer），取出前 ``buffer_size`` 個元素放入，並從緩衝區中隨機採樣，採樣後的數據用後續數據替換）；
- ``Dataset.batch(batch_size)`` ：將數據集分成批次，即對每 ``batch_size`` 個元素，使用 ``tf.stack()`` 在第0維合併，成爲一個元素；

除此以外，還有 ``Dataset.repeat()`` （重複數據集的元素）、 ``Dataset.reduce()`` （與Map相對的聚合操作）、 ``Dataset.take()`` （截取數據集中的前若干個元素）等，可參考 `API文檔 <https://www.tensorflow.org/versions/r2.0/api_docs/python/tf/data/Dataset>`_ 進一步了解。

以下以MNIST數據集進行示例。

使用 ``Dataset.map()`` 將所有圖片旋轉90度：

.. literalinclude:: /_static/code/zh/tools/tfdata/tutorial.py
    :lines: 27-37
    :emphasize-lines: 1-5

輸出

.. figure:: /_static/image/tools/mnist_1_rot90.png
    :width: 40%
    :align: center

使用 ``Dataset.batch()`` 將數據集劃分批次，每個批次的大小爲4：

.. literalinclude:: /_static/code/zh/tools/tfdata/tutorial.py
    :lines: 38-45
    :emphasize-lines: 1

輸出

.. figure:: /_static/image/tools/mnist_batch.png
    :width: 100%
    :align: center

使用 ``Dataset.shuffle()`` 將數據打散後再設置批次，緩存大小設置爲10000：

.. literalinclude:: /_static/code/zh/tools/tfdata/tutorial.py
    :lines: 47-54
    :emphasize-lines: 1

輸出

.. figure:: /_static/image/tools/mnist_shuffle_1.png
    :width: 100%
    :align: center
    
    第一次運行

.. figure:: /_static/image/tools/mnist_shuffle_2.png
    :width: 100%
    :align: center
    
    第二次運行

可見每次的數據都會被隨機打散。

.. admonition:: ``Dataset.shuffle()`` 時緩衝區大小 ``buffer_size`` 的設置

    ``tf.data.Dataset`` 作爲一個針對大規模數據設計的疊代器，本身無法方便地獲得自身元素的數量或隨機訪問元素。因此，爲了高效且較爲充分地打散數據集，需要一些特定的方法。``Dataset.shuffle()`` 採取了以下方法：

    - 設定一個固定大小爲 ``buffer_size`` 的緩衝區（Buffer）；
    - 初始化時，取出數據集中的前 ``buffer_size`` 個元素放入緩衝區；
    - 每次需要從數據集中取元素時，即從緩衝區中隨機採樣一個元素並取出，然後從後續的元素中取出一個放回到之前被取出的位置，以維持緩衝區的大小。

    因此，緩衝區的大小需要根據數據集的特性和數據排列順序特點來進行合理的設置。比如：

    - 當 ``buffer_size`` 設置爲1時，其實等價於沒有進行任何打散；
    - 當數據集的標籤順序分布極爲不均勻（例如二元分類時數據集前N個的標籤爲0，後N個的標籤爲1）時，較小的緩衝區大小會使得訓練時取出的Batch數據很可能全爲同一標籤，從而影響訓練效果。一般而言，數據集的順序分布若較爲隨機，則緩衝區的大小可較小，否則則需要設置較大的緩衝區。

.. _prefetch:

使用 ``tf.data`` 的並行化策略提高訓練流程效率
--------------------------------------------------------------------------------------

..
    https://www.tensorflow.org/guide/data_performance

當訓練模型時，我們希望充分利用計算資源，減少CPU/GPU的空載時間。然而有時，數據集的準備處理非常耗時，使得我們在每進行一次訓練前都需要花費大量的時間準備待訓練的數據，而此時GPU只能空載而等待數據，造成了計算資源的浪費，如下圖所示：

.. figure:: /_static/image/tools/datasets_without_pipelining.png
    :width: 100%
    :align: center

    常規訓練流程，在準備數據時，GPU只能空載。`1圖示來源 <https://www.tensorflow.org/guide/data_performance>`_ 。

此時， ``tf.data`` 的數據集對象爲我們提供了 ``Dataset.prefetch()`` 方法，使得我們可以讓數據集對象 ``Dataset`` 在訓練時預取出若干個元素，使得在GPU訓練的同時CPU可以準備數據，從而提升訓練流程的效率，如下圖所示：

.. figure:: /_static/image/tools/datasets_with_pipelining.png
    :width: 100%
    :align: center
    
    使用 ``Dataset.prefetch()`` 方法進行數據預加載後的訓練流程，在GPU進行訓練的同時CPU進行數據預加載，提高了訓練效率。 `2圖示來源  <https://www.tensorflow.org/guide/data_performance>`_ 。

``Dataset.prefetch()`` 的使用方法和前節的 ``Dataset.batch()`` 、 ``Dataset.shuffle()`` 等非常類似。繼續以前節的MNIST數據集爲例，若希望開啓預加載數據，使用如下代碼即可：

.. code-block:: python

    mnist_dataset = mnist_dataset.prefetch(buffer_size=tf.data.experimental.AUTOTUNE)

此處參數 ``buffer_size`` 既可手工設置，也可設置爲 ``tf.data.experimental.AUTOTUNE`` 從而由TensorFlow自動選擇合適的數值。

與此類似， ``Dataset.map()`` 也可以利用多GPU資源，並行化地對數據項進行變換，從而提高效率。以前節的MNIST數據集爲例，假設用於訓練的計算機具有2核的CPU，我們希望充分利用多核心的優勢對數據進行並行化變換（比如前節的旋轉90度函數 ``rot90`` ），可以使用以下代碼：

.. code-block:: python

    mnist_dataset = mnist_dataset.map(map_func=rot90, num_parallel_calls=2)

其運行過程如下圖所示：

.. figure:: /_static/image/tools/datasets_parallel_map.png
    :width: 100%
    :align: center

    通過設置 ``Dataset.map()`` 的 ``num_parallel_calls`` 參數實現數據轉換的並行化。上部分是未並行化的圖示，下部分是2核並行的圖示。 `3圖示來源  <https://www.tensorflow.org/guide/data_performance>`_ 。

當然，這裡同樣可以將 ``num_parallel_calls`` 設置爲 ``tf.data.experimental.AUTOTUNE`` 以讓TensorFlow自動選擇合適的數值。

除此以外，還有很多提升數據集處理性能的方式，可參考 `TensorFlow文檔 <https://www.tensorflow.org/guide/data_performance>`_ 進一步了解。後文的實例中展示了tf.data並行化策略的強大性能，可 :ref:`點此 <tfdata_performance>` 查看。

數據集元素的獲取與使用
-------------------------------------------
構建好數據並預處理後，我們需要從其中疊代獲取數據以用於訓練。``tf.data.Dataset`` 是一個Python的可疊代對象，因此可以使用For循環疊代獲取數據，即：

.. code-block:: python

    dataset = tf.data.Dataset.from_tensor_slices((A, B, C, ...))
    for a, b, c, ... in dataset:
        # 對張量a, b, c等進行操作，例如送入模型進行訓練

也可以使用 ``iter()`` 顯式創建一個Python疊代器並使用 ``next()`` 獲取下一個元素，即：

.. code-block:: python

    dataset = tf.data.Dataset.from_tensor_slices((A, B, C, ...))
    it = iter(dataset)
    a_0, b_0, c_0, ... = next(it)
    a_1, b_1, c_1, ... = next(it)

Keras支持使用 ``tf.data.Dataset`` 直接作爲輸入。當調用 ``tf.keras.Model`` 的 ``fit()`` 和 ``evaluate()`` 方法時，可以將參數中的輸入數據 ``x`` 指定爲一個元素格式爲 ``(輸入數據, 標籤數據)`` 的 ``Dataset`` ，並忽略掉參數中的標籤數據 ``y`` 。例如，對於上述的MNIST數據集，常規的Keras訓練方式是：

.. code-block:: python

    model.fit(x=train_data, y=train_label, epochs=num_epochs, batch_size=batch_size)

使用 ``tf.data.Dataset`` 後，我們可以直接傳入 ``Dataset`` ：

.. code-block:: python

    model.fit(mnist_dataset, epochs=num_epochs)

由於已經通過 ``Dataset.batch()`` 方法劃分了數據集的批次，所以這裡也無需提供批次的大小。

.. _cats_vs_dogs:

實例：cats_vs_dogs圖像分類
-------------------------------------------

以下代碼以貓狗圖片二分類任務爲示例，展示了使用 ``tf.data`` 結合 ``tf.io`` 和 ``tf.image`` 建立 ``tf.data.Dataset`` 數據集，並進行訓練和測試的完整過程。數據集可至 `這裡 <https://www.floydhub.com/fastai/datasets/cats-vs-dogs>`_ 下載。使用前須將數據集解壓到代碼中 ``data_dir`` 所設置的目錄（此處默認設置爲 ``C:/datasets/cats_vs_dogs`` ，可根據自己的需求進行修改）。

.. literalinclude:: /_static/code/zh/tools/tfdata/cats_vs_dogs.py
    :lines: 1-54
    :emphasize-lines: 13-17, 29-36, 54

使用以下代碼進行測試：

.. literalinclude:: /_static/code/zh/tools/tfdata/cats_vs_dogs.py
    :lines: 56-70

.. _tfdata_performance:

通過對以上示例進行性能測試，我們可以感受到 ``tf.data`` 的強大並行化性能。通過 ``prefetch()`` 的使用和在 ``map()`` 過程中加入 ``num_parallel_calls`` 參數，模型訓練的時間可縮減至原來的一半甚至更低。測試結果如下：

.. figure:: /_static/image/tools/tfdata_performance.jpg
    :width: 100%
    :align: center

    tf.data 的並行化策略性能測試（縱軸爲每epoch訓練所需時間，單位：秒）

.. _tfrecord:

TFRecord ：TensorFlow數據集存儲格式
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

..
    https://www.tensorflow.org/tutorials/load_data/tfrecord

TFRecord 是TensorFlow 中的數據集存儲格式。當我們將數據集整理成 TFRecord 格式後，TensorFlow就可以高效地讀取和處理這些數據集，從而幫助我們更高效地進行大規模的模型訓練。

TFRecord可以理解爲一系列序列化的 ``tf.train.Example`` 元素所組成的列表文件，而每一個 ``tf.train.Example`` 又由若干個 ``tf.train.Feature`` 的字典組成。形式如下：

::

    # dataset.tfrecords
    [
        {   # example 1 (tf.train.Example)
            'feature_1': tf.train.Feature,
            ...
            'feature_k': tf.train.Feature
        },
        ...
        {   # example N (tf.train.Example)
            'feature_1': tf.train.Feature,
            ...
            'feature_k': tf.train.Feature
        }
    ]


爲了將形式各樣的數據集整理爲 TFRecord 格式，我們可以對數據集中的每個元素進行以下步驟：

- 讀取該數據元素到內存；
- 將該元素轉換爲 ``tf.train.Example`` 對象（每一個 ``tf.train.Example`` 由若干個 ``tf.train.Feature`` 的字典組成，因此需要先建立Feature的字典）；
- 將該 ``tf.train.Example`` 對象序列化爲字符串，並通過一個預先定義的 ``tf.io.TFRecordWriter`` 寫入 TFRecord 文件。

而讀取 TFRecord 數據則可按照以下步驟：

- 通過 ``tf.data.TFRecordDataset`` 讀入原始的 TFRecord 文件（此時文件中的 ``tf.train.Example`` 對象尚未被反序列化），獲得一個 ``tf.data.Dataset`` 數據集對象；
- 通過 ``Dataset.map`` 方法，對該數據集對象中的每一個序列化的 ``tf.train.Example`` 字符串執行 ``tf.io.parse_single_example`` 函數，從而實現反序列化。

以下我們通過一個實例，展示將 :ref:`上一節 <cats_vs_dogs>` 中使用的cats_vs_dogs二分類數據集的訓練集部分轉換爲TFRecord文件，並讀取該文件的過程。

將數據集存儲爲 TFRecord 文件
-------------------------------------------

首先，與 :ref:`上一節 <cats_vs_dogs>` 類似，我們進行一些準備工作，`下載數據集 <https://www.floydhub.com/fastai/datasets/cats-vs-dogs>`_ 並解壓到 ``data_dir`` ，初始化數據集的圖片文件名列表及標籤。

.. literalinclude:: /_static/code/zh/tools/tfrecord/cats_vs_dogs.py
    :lines: 1-12

然後，通過以下代碼，疊代讀取每張圖片，建立 ``tf.train.Feature`` 字典和 ``tf.train.Example`` 對象，序列化並寫入TFRecord文件。

.. literalinclude:: /_static/code/zh/tools/tfrecord/cats_vs_dogs.py
    :lines: 14-22

值得注意的是， ``tf.train.Feature`` 支持三種數據格式：

- ``tf.train.BytesList`` ：字符串或原始Byte文件（如圖片），通過 ``bytes_list`` 參數傳入一個由字符串數組初始化的 ``tf.train.BytesList`` 對象；
- ``tf.train.FloatList`` ：浮點數，通過 ``float_list`` 參數傳入一個由浮點數數組初始化的 ``tf.train.FloatList`` 對象；
- ``tf.train.Int64List`` ：整數，通過 ``int64_list`` 參數傳入一個由整數數組初始化的 ``tf.train.Int64List`` 對象。

如果只希望保存一個元素而非數組，傳入一個只有一個元素的數組即可。

運行以上代碼，不出片刻，我們即可在 ``tfrecord_file`` 所指向的文件地址獲得一個 500MB 左右的 ``train.tfrecords`` 文件。

讀取 TFRecord 文件
-------------------------------------------

我們可以通過以下代碼，讀取之間建立的 ``train.tfrecords`` 文件，並通過 ``Dataset.map`` 方法，使用 ``tf.io.parse_single_example`` 函數對數據集中的每一個序列化的 ``tf.train.Example`` 對象解碼。

.. literalinclude:: /_static/code/zh/tools/tfrecord/cats_vs_dogs.py
    :lines: 24-36

這裡的 ``feature_description`` 類似於一個數據集的「描述文件」，通過一個由鍵值對組成的字典，告知 ``tf.io.parse_single_example`` 函數每個 ``tf.train.Example`` 數據項有哪些Feature，以及這些Feature的類型、形狀等屬性。 ``tf.io.FixedLenFeature`` 的三個輸入參數 ``shape`` 、 ``dtype`` 和 ``default_value`` （可省略）爲每個Feature的形狀、類型和默認值。這裡我們的數據項都是單個的數值或者字符串，所以 ``shape`` 爲空數組。

運行以上代碼後，我們獲得一個數據集對象 ``dataset`` ，這已經是一個可以用於訓練的 ``tf.data.Dataset`` 對象了！我們從該數據集中讀取元素並輸出驗證：

.. literalinclude:: /_static/code/zh/tools/tfrecord/cats_vs_dogs.py
    :lines: 38-43

顯示：

.. figure:: /_static/image/tools/tfrecord_cat.png
    :width: 60%
    :align: center

可見圖片和標籤都正確顯示，數據集構建成功。

.. _tffunction:

``tf.function`` ：圖執行模式 *
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

雖然默認的即時執行模式（Eager Execution）爲我們帶來了靈活及易調試的特性，但在特定的場合，例如追求高性能或部署模型時，我們依然希望使用 TensorFlow 1.X 中默認的圖執行模式（Graph Execution），將模型轉換爲高效的 TensorFlow 圖模型。此時，TensorFlow 2 爲我們提供了 ``tf.function`` 模塊，結合 AutoGraph 機制，使得我們僅需加入一個簡單的 ``@tf.function`` 修飾符，就能輕鬆將模型以圖執行模式運行。

``tf.function`` 基礎使用方法
-------------------------------------------

..
    https://www.tensorflow.org/beta/guide/autograph
    https://www.tensorflow.org/guide/autograph
    https://www.tensorflow.org/beta/tutorials/eager/tf_function
    https://pgaleone.eu/tensorflow/tf.function/2019/03/21/dissecting-tf-function-part-1/
    https://pgaleone.eu/tensorflow/tf.function/2019/04/03/dissecting-tf-function-part-2/
    https://pgaleone.eu/tensorflow/tf.function/2019/05/10/dissecting-tf-function-part-3/

在 TensorFlow 2 中，推薦使用 ``tf.function`` （而非1.X中的 ``tf.Session`` ）實現圖執行模式，從而將模型轉換爲易於部署且高性能的TensorFlow圖模型。只需要將我們希望以圖執行模式運行的代碼封裝在一個函數內，並在函數前加上 ``@tf.function`` 即可，如下例所示。關於圖執行模式的深入探討可參考 :doc:`附錄 <../appendix/static>` 。

.. warning:: 並不是任何函數都可以被 ``@tf.function`` 修飾！``@tf.function`` 使用靜態編譯將函數內的代碼轉換成計算圖，因此對函數內可使用的語句有一定限制（僅支持Python語言的一個子集），且需要函數內的操作本身能夠被構建爲計算圖。建議在函數內只使用TensorFlow的原生操作，不要使用過於複雜的Python語句，函數參數只包括TensorFlow張量或NumPy數組，並最好是能夠按照計算圖的思想去構建函數（換言之，``@tf.function`` 只是給了你一種更方便的寫計算圖的方法，而不是一顆能給任何函數加速的 `銀子彈 <https://en.wikipedia.org/wiki/No_Silver_Bullet>`_ ）。詳細內容可參考 `AutoGraph Capabilities and Limitations <https://github.com/tensorflow/tensorflow/blob/master/tensorflow/python/autograph/g3doc/reference/limitations.md>`_ 。建議配合 :doc:`附錄 <../appendix/static>` 一同閱讀本節以獲得較深入的理解。

.. literalinclude:: /_static/code/zh/model/autograph/main.py
    :emphasize-lines: 11, 18

運行400個Batch進行測試，加入 ``@tf.function`` 的程序耗時35.5秒，未加入 ``@tf.function`` 的純即時執行模式程序耗時43.8秒。可見 ``@tf.function`` 帶來了一定的性能提升。一般而言，當模型由較多小的操作組成的時候， ``@tf.function`` 帶來的提升效果較大。而當模型的操作數量較少，但單一操作均很耗時的時候，則 ``@tf.function`` 帶來的性能提升不會太大。

..
    https://www.tensorflow.org/beta/guide/autograph
    Functions can be faster than eager code, for graphs with many small ops. But for graphs with a few expensive ops (like convolutions), you may not see much speedup.

``tf.function`` 內在機制
-------------------------------------------

當被 ``@tf.function`` 修飾的函數第一次被調用的時候，進行以下操作：

- 在即時執行模式關閉的環境下，函數內的代碼依次運行。也就是說，每個 ``tf.`` 方法都只是定義了計算節點，而並沒有進行任何實質的計算。這與TensorFlow 1.X的圖執行模式是一致的；
- 使用AutoGraph將函數中的Python控制流語句轉換成TensorFlow計算圖中的對應節點（比如說 ``while`` 和 ``for`` 語句轉換爲 ``tf.while`` ， ``if`` 語句轉換爲 ``tf.cond`` 等等；
- 基於上面的兩步，建立函數內代碼的計算圖表示（爲了保證圖的計算順序，圖中還會自動加入一些 ``tf.control_dependencies`` 節點）；
- 運行一次這個計算圖；
- 基於函數的名字和輸入的函數參數的類型生成一個哈希值，並將建立的計算圖緩存到一個哈希表中。

在被 ``@tf.function`` 修飾的函數之後再次被調用的時候，根據函數名和輸入的函數參數的類型計算哈希值，檢查哈希表中是否已經有了對應計算圖的緩存。如果是，則直接使用已緩存的計算圖，否則重新按上述步驟建立計算圖。

.. hint:: 對於熟悉 TensorFlow 1.X 的開發者，如果想要直接獲得 ``tf.function`` 所生成的計算圖以進行進一步處理和調試，可以使用被修飾函數的 ``get_concrete_function`` 方法。該方法接受的參數與被修飾函數相同。例如，爲了獲取前節被 ``@tf.function`` 修飾的函數 ``train_one_step`` 所生成的計算圖，可以使用以下代碼：

    .. code-block:: python

        graph = train_one_step.get_concrete_function(X, y)

    其中 ``graph`` 即爲一個 ``tf.Graph`` 對象。

以下是一個測試題：

.. literalinclude:: /_static/code/zh/model/autograph/quiz.py
    :lines: 1-18

思考一下，上面這段程序的結果是什麼？

答案是::

    The function is running in Python
    1
    2
    2
    The function is running in Python
    0.1
    0.2    

當計算 ``f(a)`` 時，由於是第一次調用該函數，TensorFlow進行了以下操作：

- 將函數內的代碼依次運行了一遍（因此輸出了文本）；
- 構建了計算圖，然後運行了一次該計算圖（因此輸出了1）。這裡 ``tf.print(x)`` 可以作爲計算圖的節點，但Python內置的 ``print`` 則不能被轉換成計算圖的節點。因此，計算圖中只包含了 ``tf.print(x)`` 這一操作；
- 將該計算圖緩存到了一個哈希表中（如果之後再有類型爲 ``tf.int32`` ，shape爲空的張量輸入，則重複使用已構建的計算圖）。

計算 ``f(b)`` 時，由於b的類型與a相同，所以TensorFlow重複使用了之前已構建的計算圖並運行（因此輸出了2）。這裡由於並沒有真正地逐行運行函數中的代碼，所以函數第一行的文本輸出代碼沒有運行。計算 ``f(b_)`` 時，TensorFlow自動將numpy的數據結構轉換成了TensorFlow中的張量，因此依然能夠復用之前已構建的計算圖。

計算 ``f(c)`` 時，雖然張量 ``c`` 的shape和 ``a`` 、 ``b`` 均相同，但類型爲 ``tf.float32`` ，因此TensorFlow重新運行了函數內代碼（從而再次輸出了文本）並建立了一個輸入爲 ``tf.float32`` 類型的計算圖。

計算 ``f(d)`` 時，由於 ``d`` 和 ``c`` 的類型相同，所以TensorFlow復用了計算圖，同理沒有輸出文本。

而對於 ``@tf.function`` 對Python內置的整數和浮點數類型的處理方式，我們通過以下示例展現：

.. literalinclude:: /_static/code/zh/model/autograph/quiz.py
    :lines: 18-24

結果爲::

    The function is running in Python
    1
    The function is running in Python
    2
    1
    The function is running in Python
    0.1
    The function is running in Python
    0.2
    0.1

簡而言之，對於Python內置的整數和浮點數類型，只有當值完全一致的時候， ``@tf.function`` 才會復用之前建立的計算圖，而並不會自動將Python內置的整數或浮點數等轉換成張量。因此，當函數參數包含Python內置整數或浮點數時，需要格外小心。一般而言，應當只在指定超參數等少數場合使用Python內置類型作爲被 ``@tf.function`` 修飾的函數的參數。

..
    https://www.tensorflow.org/versions/r2.0/api_docs/python/tf/function
    Note that unlike other TensorFlow operations, we don't convert python numerical inputs to tensors. Moreover, a new graph is generated for each distinct python numerical value, for example calling g(2) and g(3) will generate two new graphs (while only one is generated if you call g(tf.constant(2)) and g(tf.constant(3))). Therefore, python numerical inputs should be restricted to arguments that will have few distinct values, such as hyperparameters like the number of layers in a neural network. This allows TensorFlow to optimize each variant of the neural network.

下一個思考題：

.. literalinclude:: /_static/code/zh/model/autograph/quiz_2.py

這段代碼的輸出是::

    tf.Tensor(1.0, shape=(), dtype=float32)
    tf.Tensor(2.0, shape=(), dtype=float32)
    tf.Tensor(3.0, shape=(), dtype=float32)

正如同正文裡的例子一樣，你可以在被 ``@tf.function`` 修飾的函數裡調用 ``tf.Variable`` 、 ``tf.keras.optimizers`` 、 ``tf.keras.Model`` 等包含有變量的數據結構。一旦被調用，這些結構將作爲隱含的參數提供給函數。當這些結構內的值在函數內被修改時，在函數外也同樣生效。

AutoGraph：將Python控制流轉換爲TensorFlow計算圖
--------------------------------------------------------------------------------------

前面提到，``@tf.function`` 使用名爲AutoGraph的機制將函數中的Python控制流語句轉換成TensorFlow計算圖中的對應節點。以下是一個示例，使用 ``tf.autograph`` 模塊的低層API ``tf.autograph.to_code`` 將函數 ``square_if_positive`` 轉換成TensorFlow計算圖：

.. literalinclude:: /_static/code/zh/model/autograph/autograph.py

輸出：

::

    tf.Tensor(1, shape=(), dtype=int32) tf.Tensor(0, shape=(), dtype=int32)
    def tf__square_if_positive(x):
        do_return = False
        retval_ = ag__.UndefinedReturnValue()
        cond = x > 0

        def get_state():
            return ()

        def set_state(_):
            pass

        def if_true():
            x_1, = x,
            x_1 = x_1 * x_1
            return x_1

        def if_false():
            x = 0
            return x
        x = ag__.if_stmt(cond, if_true, if_false, get_state, set_state)
        do_return = True
        retval_ = x
        cond_1 = ag__.is_undefined_return(retval_)

        def get_state_1():
            return ()

        def set_state_1(_):
            pass

        def if_true_1():
            retval_ = None
            return retval_

        def if_false_1():
            return retval_
        retval_ = ag__.if_stmt(cond_1, if_true_1, if_false_1, get_state_1, set_state_1)
        return retval_

我們注意到，原函數中的Python控制流 ``if...else...`` 被轉換爲了 ``x = ag__.if_stmt(cond, if_true, if_false, get_state, set_state)`` 這種計算圖式的寫法。AutoGraph起到了類似編譯器的作用，能夠幫助我們通過更加自然的Python控制流輕鬆地構建帶有條件/循環的計算圖，而無需手動使用TensorFlow的API進行構建。

使用傳統的 ``tf.Session`` 
------------------------------------------- 

不過，如果你依然鍾情於TensorFlow傳統的圖執行模式也沒有問題。TensorFlow 2 提供了 ``tf.compat.v1`` 模塊以支持TensorFlow 1.X版本的API。同時，只要在編寫模型的時候稍加注意，Keras的模型是可以同時兼容即時執行模式和圖執行模式的。注意，在圖執行模式下， ``model(input_tensor)`` 只需運行一次以完成圖的建立操作。

例如，通過以下代碼，同樣可以在MNIST數據集上訓練前面所建立的MLP或CNN模型：

.. literalinclude:: /_static/code/zh/model/mnist/main.py
    :lines: 112-136


關於圖執行模式的更多內容可參見 :doc:`../appendix/static`。

``tf.TensorArray`` ：TensorFlow 動態數組 *
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

..
    https://www.tensorflow.org/api_docs/python/tf/TensorArray

在部分網絡結構，尤其是涉及到時間序列的結構中，我們可能需要將一系列張量以數組的方式依次存放起來，以供進一步處理。當然，在即時執行模式下，你可以直接使用一個Python列表（List）存放數組。不過，如果你需要基於計算圖的特性（例如使用 ``@tf.function`` 加速模型運行或者使用SavedModel導出模型），就無法使用這種方式了。因此，TensorFlow提供了 ``tf.TensorArray`` ，一種支持計算圖特性的TensorFlow動態數組。

其聲明的方式爲：

- ``arr = tf.TensorArray(dtype, size, dynamic_size=False)`` ：聲明一個大小爲 ``size`` ，類型爲 ``dtype`` 的TensorArray ``arr`` 。如果將 ``dynamic_size`` 參數設置爲 ``True`` ，則該數組會自動增長空間。

其讀取和寫入的方法爲：

- ``write(index, value)`` ：將 ``value`` 寫入數組的第 ``index`` 個位置；
- ``read(index)`` ：讀取數組的第 ``index`` 個值；

除此以外，TensorArray還包括 ``stack()`` 、 ``unstack()`` 等常用操作，可參考 `文檔 <https://www.tensorflow.org/api_docs/python/tf/TensorArray>`_ 以了解詳情。

請注意，由於需要支持計算圖， ``tf.TensorArray`` 的 ``write()`` 方法是不可以忽略左值的！也就是說，在圖執行模式下，必須按照以下的形式寫入數組：

.. code-block:: python

    arr = arr.write(index, value)

這樣才可以正常生成一個計算圖操作，並將該操作返回給 ``arr`` 。而不可以寫成：

.. code-block:: python

    arr.write(index, value)     # 生成的計算圖操作沒有左值接收，從而丟失

一個簡單的示例如下：

.. literalinclude:: /_static/code/zh/tools/tensorarray/example.py

輸出：

::
    
    tf.Tensor(0.0, shape=(), dtype=float32) tf.Tensor(1.0, shape=(), dtype=float32) tf.Tensor(2.0, shape=(), dtype=float32)

``tf.config``：GPU的使用與分配 *
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

..
    https://www.tensorflow.org/beta/guide/using_gpu

指定當前程序使用的GPU
------------------------------------------- 

很多時候的場景是：實驗室/公司研究組裡有許多學生/研究員需要共同使用一台多GPU的工作站，而默認情況下TensorFlow會使用其所能夠使用的所有GPU，這時就需要合理分配顯卡資源。

首先，通過 ``tf.config.list_physical_devices`` ，我們可以獲得當前主機上某種特定運算設備類型（如 ``GPU`` 或 ``CPU`` ）的列表，例如，在一台具有4塊GPU和一個CPU的工作站上運行以下代碼：

.. code-block:: python

    gpus = tf.config.list_physical_devices(device_type='GPU')
    cpus = tf.config.list_physical_devices(device_type='CPU')
    print(gpus, cpus)

輸出：

.. code-block:: python

    [PhysicalDevice(name='/physical_device:GPU:0', device_type='GPU'), 
     PhysicalDevice(name='/physical_device:GPU:1', device_type='GPU'), 
     PhysicalDevice(name='/physical_device:GPU:2', device_type='GPU'), 
     PhysicalDevice(name='/physical_device:GPU:3', device_type='GPU')]     
    [PhysicalDevice(name='/physical_device:CPU:0', device_type='CPU')]

可見，該工作站具有4塊GPU：``GPU:0`` 、 ``GPU:1`` 、 ``GPU:2`` 、 ``GPU:3`` ，以及一個CPU ``CPU:0`` 。

然後，通過 ``tf.config.set_visible_devices`` ，可以設置當前程序可見的設備範圍（當前程序只會使用自己可見的設備，不可見的設備不會被當前程序使用）。例如，如果在上述4卡的機器中我們需要限定當前程序只使用下標爲0、1的兩塊顯卡（``GPU:0`` 和 ``GPU:1``），可以使用以下代碼：

.. code-block:: python

    gpus = tf.config.list_physical_devices(device_type='GPU')
    tf.config.set_visible_devices(devices=gpus[0:2], device_type='GPU')

.. tip:: 使用環境變量 ``CUDA_VISIBLE_DEVICES`` 也可以控制程序所使用的GPU。假設發現四卡的機器上顯卡0,1使用中，顯卡2,3空閒，Linux終端輸入::

        export CUDA_VISIBLE_DEVICES=2,3

    或在代碼中加入

    .. code-block:: python

        import os
        os.environ['CUDA_VISIBLE_DEVICES'] = "2,3"

    即可指定程序只在顯卡2,3上運行。

設置顯存使用策略
------------------------------------------- 

默認情況下，TensorFlow將使用幾乎所有可用的顯存，以避免內存碎片化所帶來的性能損失。不過，TensorFlow提供兩種顯存使用策略，讓我們能夠更靈活地控制程序的顯存使用方式：

- 僅在需要時申請顯存空間（程序初始運行時消耗很少的顯存，隨著程序的運行而動態申請顯存）；
- 限制消耗固定大小的顯存（程序不會超出限定的顯存大小，若超出的報錯）。

可以通過 ``tf.config.experimental.set_memory_growth`` 將GPU的顯存使用策略設置爲「僅在需要時申請顯存空間」。以下代碼將所有GPU設置爲僅在需要時申請顯存空間：

.. code-block:: python

    gpus = tf.config.list_physical_devices(device_type='GPU')
    for gpu in gpus:
        tf.config.experimental.set_memory_growth(device=gpu, enable=True)

以下代碼通過 ``tf.config.set_logical_device_configuration`` 選項並傳入 ``tf.config.LogicalDeviceConfiguration`` 實例，設置TensorFlow固定消耗 ``GPU:0`` 的1GB顯存（其實可以理解爲建立了一個顯存大小爲1GB的「虛擬GPU」）：

.. code-block:: python

    gpus = tf.config.list_physical_devices(device_type='GPU')
    tf.config.set_logical_device_configuration(
        gpus[0],
        [tf.config.LogicalDeviceConfiguration(memory_limit=1024)])

.. hint:: TensorFlow 1.X 的 圖執行模式 下，可以在實例化新的session時傳入 ``tf.compat.v1.ConfigPhoto`` 類來設置TensorFlow使用顯存的策略。具體方式是實例化一個 ``tf.ConfigProto`` 類，設置參數，並在創建 ``tf.compat.v1.Session`` 時指定Config參數。以下代碼通過 ``allow_growth`` 選項設置TensorFlow僅在需要時申請顯存空間：

    .. code-block:: python

        config = tf.compat.v1.ConfigProto()
        config.gpu_options.allow_growth = True
        sess = tf.compat.v1.Session(config=config)

    以下代碼通過 ``per_process_gpu_memory_fraction`` 選項設置TensorFlow固定消耗40%的GPU顯存：

    .. code-block:: python

        config = tf.compat.v1.ConfigProto()
        config.gpu_options.per_process_gpu_memory_fraction = 0.4
        tf.compat.v1.Session(config=config)

單GPU模擬多GPU環境
-------------------------------------------

當我們的本地開發環境只有一個GPU，但卻需要編寫多GPU的程序在工作站上進行訓練任務時，TensorFlow爲我們提供了一個方便的功能，可以讓我們在本地開發環境中建立多個模擬GPU，從而讓多GPU的程序調試變得更加方便。以下代碼在實體GPU ``GPU:0`` 的基礎上建立了兩個顯存均爲2GB的虛擬GPU。

.. code-block:: python

    gpus = tf.config.list_physical_devices('GPU')
    tf.config.set_logical_device_configuration(
        gpus[0],
        [tf.config.LogicalDeviceConfiguration(memory_limit=2048),
         tf.config.LogicalDeviceConfiguration(memory_limit=2048)])

我們在 :ref:`單機多卡訓練 <multi_gpu>` 的代碼前加入以上代碼，即可讓原本爲多GPU設計的代碼在單GPU環境下運行。當輸出設備數量時，程序會輸出：

::

    Number of devices: 2

.. raw:: html

    <script>
        $(document).ready(function(){
            $(".rst-footer-buttons").after("<div id='discourse-comments'></div>");
            DiscourseEmbed = { discourseUrl: 'https://discuss.tf.wiki/', topicId: 191 };
            (function() {
                var d = document.createElement('script'); d.type = 'text/javascript'; d.async = true;
                d.src = DiscourseEmbed.discourseUrl + 'javascripts/embed.js';
                (document.getElementsByTagName('head')[0] || document.getElementsByTagName('body')[0]).appendChild(d);
            })();
        });
    </script>