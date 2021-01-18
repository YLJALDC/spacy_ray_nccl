# spacy_ray_collective
This github is inherited from spacy-ray (dev branch). It trains model using the new collective calls available in current ray-1.2-dev, <br />
replacing the original get(), set() in ray. <br />
The runtime comparison for 1000 update using spacy pipeline = ["tok2vec", "ner"]: <br />
    | Comparison    | spacy-ray     | spacy-ray-nccl |  ratio  |  
    | ------------- | ------------- | -------------- | ------- | 
    | 1 worker      | 137.5 ± 2.1   | 125.8 ± 2.0    |  1.09x  |
    | 2 workers     | 354.1 ± 16.8  | 184.2 ± 1.28   |  1.92x  |  
    | 4 workers     | 523.9 ± 10.4  | 198.5 ± 0.85   |  2.64x  |  
    | 8 workers     | 710.1 ± 3.0   | 228.5 ± 1.60   |  3.11x  | 
    | 16 workers    | 1296.1 ± 42.1 | 273.7 ± 6.93   |  4.74x  | 

Mean and standard deviation obtained by three trials (in seconds).  <br />
<br />
Runtime comparison: The ideal plot should be a horizontal line.
![runtime](results/time_comparison.png)
Speedup comparison: The trivial speedup is a horizontal line y = 1.
![speedup](results/ratio_comparison.png)
    

To install the necessary module: <br />
   &nbsp;  1. ```conda create -n spacy-ray python=3.7.3``` <br />
    2. ```conda activate spacy-ray``` <br />
    3. ```pip install spacy-nightly[cuda]``` <br />
       - This will take some time, if observe a build error in cupy, try: ```pip install cupy-cuda[version]``` (e.g. for cudatoolkit 11.0: pip install cupy-cuda110 <br />
    4. ```pip install spacy-ray``` <br />
       - Run     ```python -m spacy ray --help```     to check this module is installed correctly <br />
    5. The collective calls are only available in current ray github. Instead we use the latest ray-1.1 in pip to test runtime. <br />
       - Get collective code:     ```git clone https://github.com/ray-project/ray``` <br />
       - access the installed code of ray 1.1:    ```cd [path-to-packages]/ray``` <br />
         If using conda, typically the path would be ```[path-to-conda]/anaconda3/envs/spacy-ray/lib/python3.7/site-packages/``` <br />
       - copy the code over: ```cp -r [path-to-github-ray]/python/ray/util/collective [path-to-ray]/ray/util``` <br />
       - add to __init__: ```vim [path-to-ray]/ray/util/__init__.py``` -> from ray.util import ray, append "collective" to the __all__ dict. <br />
    6. The last step is to replace the installed spacy-ray using this github. <br />
       - ```git clone https://github.com/YLJALDC/spacy_ray_nccl``` <br />
       - ```mv [path-to-github-spacy-ray-nccl] [path-to-packages]``` <br />
       - make a copy of the original spacy_ray in case you would like to recover the comparison:  ```mv [path-to-packages]/spacy_ray [path-to-packages]/spacy_ray_original``` <br />
       - ```mv [path-to-packages]/spacy_ray_nccl [path-to-packages]/spacy_ray``` <br />

To run examples: <br />
    1. ```git clone https://github.com/YLJALDC/spacy_ray_example``` <br />
    2. ```cd spacy_ray_example/tmp/experiments/en-ent-wiki``` <br />
    3. Setup the ray cluster in different machine. The code will detect the available ray cluster and attach. <br />
    4. Modify the config (for training hyperparameter) and project.yml (for number of workers) <br />
    5. Download and process necessary files: (reference: https://github.com/explosion/spacy-ray/tree/develop/tmp/experiments/en-ent-wiki) <br />
        - ```spacy project assets``` <br />
        - ```spacy project run corpus``` <br />
    6. spacy project run ray-train <br />

Evaluation note: <br \>
    The github turns off the score evaluation for comparison (because evaluation takes a long time, we only want to measure the speedup during training). <br />
    To turn on: comment the hard-coded socres in train() function at worker.py, change the if condition from if self.rank ==0 to if True, and uncomment socres = self.evaluate() <br />

Implementation note: <br />
    The original spacy-ray uses sharded parameter server and update the model parameter in each worker asynchronizely. The current implementation uses all_reduce strategy, which <br />
performs similarly to a sharded parameter server. It has a whole copy of the model parameter in each worker. For update, it uses collective.allreduce() to synchronize gradients. <br />

Ray cluster setup node:  <br />
    A template for setting up a 16 machine ray cluster: <br />
```
  1 #!/bin/bash 
  2 
  3 MY_IPADDR=$(hostname -i) 
  4 echo $MY_IPADDR 
  5 
  6 ray stop --force 
  7 sleep 3 
  8 ray start --head --port=6380 --object-manager-port=8076  --object-store-memory=32359738368 
  9 sleep 2 
 10 
 11 for i in {1..15} 
 12 do 
 13   echo "=> node $i" 
 14   ssh -o StrictHostKeyChecking=no h$i.ray-dev-16.BigLearning "cd spacy_ray_example/tmp/experiments/en-ent-wiki;  source ~/anaconda3/bin/activate; conda activate spacy-ray; ray stop --force; ray start --address='$MY_IPADDR:6380' --object-manager-port=8076 --object-store-memory=32359738368"; 
 15 done 
 16 wait 
```
    
