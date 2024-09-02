---
title: Neural network to detect network botnet traffic
date: '2020-07-05'
summary: 
---

Goal
====

In this post, I will summarise a project I made for my master in [Cybersecurity for UNED](http://portal.uned.es/portal/page?_pageid=93,69878480&_dad=portal).

Our goal is to be able to detect Botnet traffic.

Using Keras to detect Botnet traffic
====================================

Keras is a perfect tool for Machine Learning experts and other developers alike. It can be as complicated as you want to make it, or as simple as you need it to be.

Chose data
----------

One of the most important decisions when attempting a project like so from the ground up is to choose or create the dataset you are going to use in the project wisely.

The quality of this decision will be strongly correlated with the quality of the final Neural Network model.

In this project, I have chosen to use a [dataset: ISOT HTTP Botnet Database](https://drive.google.com/open?id=1LW-FNhgqTZfYswHSLPUxtwccwM75O4J4) created by the Victoria University in 2017. This dataset consists of a wide variety of Botnet traffic, along with samples of benign traffic.

The dataset in it’s totality consists of approximately eleven million captured web packets. The dataset network topology:

![](/images/Botnet_Network_structure.png)

Selecting properties
--------------------

Once we have decided the dataset we are going to work with, we will now begin selecting the properties we will include in our study.

Our mail goal is to detect Botnet network packets traveling through our network, so we are going to be interested in the following properties:

*   `ip.src (Categorical)`
*   `ip.dst (Categorical)`
*   `_ws.col.Protocol (Categorical)`
*   `_ws.col.Info (Categorical)`
*   `frame.len (Continuous)`

Data conversion to `.csv`
-------------------------

To extract the following data from the `pcap` files, we are going to use a command-line utility called _tshark_.

In our case, as we want to merge all the data captured, we have developed a script that merges the converted `.csv` files together.

    #!/bin/bash
    
    # Check for the first log.
    # We have to remove subsequent headers.
    first=true
    
    # Concatenate all the malign files
    for file in ./botnet_data/*.pcap; do
      echo "Processing $file"
      if $first; then
        first=false
        tshark -r $file -T fields -e frame.time_epoch -e ip.src -e ip.dst -e _ws.col.Protocol -e frame.len -e _ws.col.Info -E separator=, -E header=y > network_malign_traffic.csv
      else
        tshark -r $file -T fields -e frame.time_epoch -e ip.src -e ip.dst -e _ws.col.Protocol -e frame.len -e _ws.col.Info -E separator=, >> network_malign_traffic.csv
      fi
    done
    
    head -n 1 network_malign_traffic.csv > merged_network_malign_traffic.csv
    tail -n +2 network_malign_traffic.csv >> merged_network_malign_traffic.csv
    
    first=true
    
    # Concatenate all the benign files
    for file in ./application_data/*.pcap; do
      echo "Processing $file"
      if $first; then
        first=false
        tshark -r $file -T fields -e frame.time_epoch -e ip.src -e ip.dst -e _ws.col.Protocol -e frame.len -e _ws.col.Info -E separator=, -E header=y > network_benign_traffic.csv
      else
        tshark -r $file -T fields -e frame.time_epoch -e ip.src -e ip.dst -e _ws.col.Protocol -e frame.len -e _ws.col.Info -E separator=, >> network_benign_traffic.csv
      fi
    done
    
    head -n 1 network_benign_traffic.csv > merged_network_benign_traffic.csv
    tail -n +2 network_benign_traffic.csv  >> merged_network_benign_traffic.csv
    

It is important that we separate captures from Botnet and benign traffic until we have added to each row it’s Botnet class. This will be the property the neural network will compare it’s result to in order to generate the weights.

Sanitize data
-------------

Following the above conversion, we are now going to clean the dataset.

In our case, we have instances of malformed rows, where there is a excess of `,` separators, values that are not entered, and invalid data in properties.

To clean the dataset, we will use Python.

    import sys
    import re
    import pandas as pd 
    import fileinput
    
    def delete_extra_commas(file_name):
        for line in fileinput.input(file_name, inplace=True):
            character_count = 0
            for character in line:
                if character == ',': character_count = character_count + 1
                if character == ',' and character_count > 5:
                    continue
                else:
                    print('{}'.format(character), end='')
    
    def delete_invalid_entries(file_name):
        #df = pd.read_csv(file_name, header=0)
        df = pd.read_csv(file_name, header=0,
                    dtype={'frame.time_epoch': float,
                           'ip.src': str,
                           'ip.dst': str,
                           '_ws.col.Protocol': str,
                           'frame.len': str,
                           '_ws.col.Info': str,
                           'Botnet': str})
        ip_addr = re.compile(r"^\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}$")
        bad_rows = []
        i = 0
        while i < len(df):
            if re.match(ip_addr, df['frame.len'][i]):
                bad_rows.append(i)
            i = i + 1
        print("Deleting bad rows: ", bad_rows)
        df.drop(df.index[bad_rows], inplace=True)
        df.to_csv(file_name, index=False)
    
    def remove_null_values(file_name):
        for line in fileinput.input(file_name, inplace=True):
            previous_character_is_comma = False
            for character in line:
                if character == ',' and previous_character_is_comma:
                    print('{}'.format('UNKNOWN'), end='')
                previous_character_is_comma = False
                print('{}'.format(character), end='')
                if character == ',': previous_character_is_comma = True     
    
    If len(sys.argv) == 1:
        print("Usage: sanitize.py {file}")
        exit()
    
    for file_name in sys.argv[1:]:
        print("Processing ", file_name)
        delete_extra_commas(file_name)
        remove_null_values(file_name)
        delete_invalid_entries(file_name)
    
    

This generates a valid, sane dataset. The last step needed in order to be used with our model is to append the `Botnet` class to every row.

Add `Botnet` Class and order
----------------------------

    #!/bin/bash
    
    # Add a new column, indicating if it is a botnet or not
    echo "Adding y column"
    awk -F"," 'BEGIN { OFS = "," } {$7="0"; print}' merged_network_benign_traffic.csv > botnet_identified_network_benign_traffic.csv
    awk -F"," 'BEGIN { OFS = "," } {$7="1"; print}' merged_network_malign_traffic.csv > botnet_identified_network_malign_traffic.csv
    
    # Replace the end of the first line with a "Botnet" label instead of a 1
    sed -i '1s/0$/Botnet/' botnet_identified_network_benign_traffic.csv
    sed -i '1s/1$/Botnet/' botnet_identified_network_malign_traffic.csv
    
    # Merge both files into one 
    echo "Merging files into merged_traffic.csv"
    head -n 1 botnet_identified_network_malign_traffic.csv > merged_traffic.csv
    tail -n +2 botnet_identified_network_benign_traffic.csv >> merged_traffic.csv
    tail -n +2 botnet_identified_network_malign_traffic.csv >> merged_traffic.csv
    
    # sort on epoch_time
    echo "Sorting entries into sorted_merged_traffic.csv"
    head -n 1 merged_traffic.csv > sorted_merged_traffic.csv
    tail -n +2 merged_traffic.csv | sort -k 1 >> sorted_merged_traffic.csv
    

Split data
----------

Now our dataset is prepared for analysis, we have to split it in two. We will train our model with one part, and test our model with another.

The goal of this approach is to test the generated model with data we have not used to train it. This mitigates the possibility of over fitting data.

In our case, we have split the dataset in 70% Train, 30% Test.

Train model
-----------

During the training phase of the model, we went through several models, including hidden layers, number of neurones per layer, activation functions and all.

    model = Sequential()
    model.add(Dense(1, input_dim=X_train.shape[1], activation='relu', kernel_initializer='he_normal'))
    model.add(Dense(1, activation='sigmoid'))
    
    # compile the keras model
    model.compile(loss='binary_crossentropy', optimizer='adam', metrics=['accuracy'])
    # fit the keras model on the dataset
    
    # Batch gradient descent
    # The number of batches is equal to the number of training set
    model.fit(X_train, y_train, epochs=15, batch_size=X_train.shape[1], verbose=2)
    

Due to the nature of the data selected, the conclusion we have come to is that the dataset can be linearly separated. This explains why we don’t need a hidden layer in our model.

Save model
----------

This is a key step in the model generation. If we don’t save the model, we have to recreate on every time we want to use it.

    model.save('modeloSecuencial.h5')
    

Model results
-------------

Even with this extremely simple model, we have managed to achieve a high percentage of Botnet detection model.

    _________________________________________________________________
    Layer (type)                 Output Shape              Param #
    =================================================================
    dense_1 (Dense)              (None, 1)                 678
    _________________________________________________________________
    dense_2 (Dense)              (None, 1)                 2
    =================================================================
    Total params: 680
    Trainable params: 680
    Non-trainable params: 0
    _________________________________________________________________
    Compiling the Keras model
    Fitting the Keras model
    Epoch 1/15
     - 1s - loss: 0.6967 - accuracy: 0.0812
    Epoch 2/15
     - 1s - loss: 0.6957 - accuracy: 0.4601
    Epoch 3/15
     - 1s - loss: 0.6946 - accuracy: 0.4885
    Epoch 4/15
     - 1s - loss: 0.6934 - accuracy: 0.5074
    Epoch 5/15
     - 1s - loss: 0.6923 - accuracy: 0.5819
    Epoch 6/15
     - es - loss: 0.6911 - accuracy: 0.6698
    Epoch 7/15
     - 1s - loss: 0.6900 - accuracy: 0.8539
    Epoch 8/15
     - 1s - loss: 0.6889 - accuracy: 0.9269
    Epoch 9/15
     - 1s - loss: 0.6878 - accuracy: 0.9499
    Epoch 10/15
     - 1s - loss: 0.6867 - accuracy: 0.9540
    .
    . (Comentado por brevedad)
    .
    Epoch 15/15
     - 1s - loss: 0.6815 - accuracy: 0.9540
    Model: "sequential_1"
    _________________________________________________________________
    Layer (type)                 Output Shape              Param #
    =================================================================
    dense_1 (Dense)              (None, 1)                 678
    _________________________________________________________________
    dense_2 (Dense)              (None, 1)                 2
    =================================================================
    Total params: 680
    Trainable params: 680
    Non-trainable params: 0
    _________________________________________________________________
    evaluating the Keras model
    Accuracy: 94.32
    Model saved to disk
    

Further work
------------

Once we have generated the model we are happy with, the next step is to implement the infrastructure needed to support the given model.

Once we have our static model, we can utilize it to dynamically analyze network traffic and perform certain action if Botnet traffic is detected. This is called: **Intrusion Prevention System**.

Conclusion
==========

I realized the most difficult part of this project is by far the data acquisition and validation phase.

Another key step is investigating the properties and benefits of every model modifier. Be it activation functions, optimizers, or number of hidden layers and neurones.
