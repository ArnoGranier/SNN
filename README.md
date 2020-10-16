# SNN
## [Python 3.6, Tensorflow] Spiking Neural Network Lib

<object data="https://github.com/ArnoGranier/SNN/files/2010589/ter.pdf" type="application/pdf" width="700px" height="700px">
    <embed src="https://github.com/ArnoGranier/SNN/files/2010589/ter.pdf">
        Download main PDF : <a href="https://github.com/ArnoGranier/SNN/files/2010589/ter.pdf">Download PDF</a> or go to ter.pdf to view it (IN FRENCH).</p>
    </embed>
</object>


Contact : arno@ini.uzh.ch

### Simple example :
_Suppose we want to simulate a fully-interconnected network of 1000 izhikievich-neurons_

First we need to import snn.network : 
```
import numpy as np
import tensorflow as tf
import snn.network as snn
```

Then we need to specify some parameters.

The time step :
```
dt = 0.1
```

The size of the population :
```
size = 1000
```

The decay function, delay and time window of the synapse model (see ter.pdf II.1.2 (in french) or (Roth, A., & van Rossum, M. C. (2009). Modeling synapses. Computational modeling methods for neuroscientists, 6, 139-160.) for more details) :
```
decay = lambda t: np.exp( - (t / 20)) ; delay = 5 ; howfar = 50
```

The external input (which could also be defined as a function of time) :
```
external_input = 5
```

The parameters of the Izhikievich model, here they are set to modelize Regular Spiking neurons (see ter.pdf I.3.3 (in french) or (Izhikevich, E. M. (2003). Simple model of spiking neurons. IEEE Transactions on neural networks, 14(6), 1569-1572.) for explanation on the model and sets of parameters for other type of neuron behaviour) :
```
parameters = {'a': 0.02, 'b': 0.2, 'c': -65, 'd': 8}
```

The weights matrix W = (wij) where wij is the weight of the connexion from the ith neuron to the jth neuron. Here we take each weight to be a random number generated by a N(0,1).
```
W = np.random.normal(0, 1, size=(size, size))
```

We can then build the tensorflow graph for our model using snn.network. We first build a empty tensorflow graph, then we create an Izhi_Nucleus (which is a python object that stock some data in convenient way), then we connect this Izhi_Nucleus with itself using weight matrix W and the function, delay and time window specified earlier (this just update the list of afference list of the Izhi_Nucleus), and finaly we fill the tensorflow graph with the necessary operations to simulate our model.
```{r, engine='python', count_lines}
graph = tf.Graph()
with graph.as_default():
    N = snn.Izhi_Nucleus(size, label='N', **parameters, Iext=external_input)
    snn.connect(N, N, W, delay=delay, decay=decay, howfar=howfar)
    data = snn.build_izhi(dt, [N, ], synapse_type='simple')
```

We will then simulate our model. To do so, we first need to specify the duration of the simulation.
```
T = 1000
```

We then simulate the model via snn.network.
```
snn.simulate(T, dt, graph, [N, ], data)
```

And finally we can plot some data.

For example we can do a raster plot
```
snn.raster_plot(N)
snn.show()
```

![alt tag](https://user-images.githubusercontent.com/27825602/40060165-469fbb02-5856-11e8-9d02-c55ed2a1e847.png)

And we can plot the variables v and I for the first neuron of the nucleus
We can plot a raster plot of our nucleus by doing
```
snn.plot_neuron_by_idx(T, dt, {N:0}, variables=['v',])
snn.show()
```

![alt tag](https://user-images.githubusercontent.com/27825602/40059853-7b51dc6e-5855-11e8-9c4d-c08fd0addf3c.png)

The complete code :
```
import numpy as np
import tensorflow as tf
import snn.network as snn

dt = 0.1
size = 1000
decay = lambda t: np.exp(- (t / 20)) ; delay = 5 ; howfar = 50
external_input = 5
parameters = {'a': 0.02, 'b': 0.2, 'c': -65, 'd': 8}
W = np.random.normal(0, 1, size=(size, size))
graph = tf.Graph()
with graph.as_default():
    N = snn.Izhi_Nucleus(size, label='N', **parameters, Iext=external_input)
    snn.connect(N, N, W, delay=delay, decay=decay, howfar=howfar)
    data = snn.build_izhi(dt, [N, ], synapse_type='simple')

T = 300
snn.simulate(T, dt, graph, [N, ], data)
snn.raster_plot(N)
snn.plot_neuron_by_idx(T, dt, {N:0}, variables=['v', 'I'])
snn.show()
```

### A more complex example :
_We provide an example of the implementation of a more complex ad-hoc model of the basal ganglia_ (see ter.pdf (in french) or 

__(__-Bahuguna, J., Tetzlaff, T., Kumar, A., Hellgren Kotaleski, J., & Morrison, A. (2017). Homologous Basal Ganglia Network Models in Physiological and Parkinsonian Conditions. Frontiers in computational neuroscience, 11, 79.__)__, 

__(__ Humphries, M. D., Stewart, R. D., & Gurney, K. N. (2006). A physiologically plausible model of action selection and oscillatory activity in the basal ganglia. Journal of Neuroscience, 26 (50), 12921–12942. __)__ and

__(__ Héricé, C., Khalil, R., Moftah, M., Boraud, T., Guthrie, M., & Garenne, A. (2016). Decision making under uncertainty in a spiking neural network model of the basal ganglia. Journal of integrative neuroscience, 15 (04), 515–538. __)__ 

from wich general architecture and parameters (weights and delays) are extracted.)

![alt tag](https://user-images.githubusercontent.com/27825602/40059873-8429a3d0-5855-11e8-8c47-4e565d9b4b87.JPG)

__THE GOAL IS TO DEMONSTRATE THE UTILITY OF SNN.NETWORK, NOT TO PRODUCE A COMPLETE AND PROOF-TESTED MODEL OF THE BASAL GANGLIA !__ 

To implement such model with snn.network, one could do as follow :

```
from snn.network import *
import numpy as np
import tensorflow as tf
np.random.seed(123)
tf.set_random_seed(123)

T, dt = 1000, 0.2

# Size of populations
names = ['CTX','D1','D2','Gpi','TA','TI','STN']
sizes = [2000 , 400, 400, 200 , 300, 300, 200]

# parameters
# CTX, Gpi -> Regular spiking
# D1, D2, TA, TI -> Fast spiking
# STN -> Intrinsically bursting
parameters = [
              {'a':0.02, 'b':0.2, 'c':-65, 'd':8}, #CTX
              {'a':0.1, 'b':0.2, 'c':-65, 'd':2}, #D1
              {'a':0.1, 'b':0.2, 'c':-65, 'd':2}, #D2
              {'a':0.02, 'b':0.2, 'c':-65, 'd':8}, #Gpi
              {'a':0.1, 'b':0.2, 'c':-65, 'd':2}, #TA
              {'a':0.1, 'b':0.2, 'c':-65, 'd':2}, #TI
              {'a':0.02, 'b':0.2, 'c':-55, 'd':4}, #STN
             ]

# connexion matrix
connexion_matrix = [#CTX D1  D2  Gpi TA  TI  STN
                    [0, 30,  6,   0,  0,  0, 30], #CTX
                    [0, -2, -2, -10,  0,  0,  0], #D1
                    [0, -2, -3,   0, -2, -2,  0], #D2
                    [0,  0,  0,   0,  0,  0,  0], #Gpi
                    [0, -2, -2,   0, -2, -2, -2], #TA
                    [0, -2, -2,  -2, -2, -2, -4], #TI
                    [0,  0,  0,  20, 20, 20,  0]  #STN
                   ] 
                   
# delay matrix
delay_matrix =     [#CTX  D1    D2   Gpi   TA    TI    STN
                    [0,   10,   10,    0,    0,    0,  2.5], #CTX
                    [0,   14,   10,    7,    0,    0,    0], #D1
                    [0,   11,   13,    0,    5,    5,    0], #D2
                    [0,    0,    0,    0,    0,    0,    0], #Gpi
                    [0,    6,    6,    0,    1,    1,    4], #TA
                    [0,    6,    6,    3,    1,    1,    4], #TI
                    [0,    0,    0,  1.5,    2,    2,    0]  #STN
                   ] 

# Decay of excitatory syanapses
decay_p = lambda t: np.exp(-t / 20) ; howfar_p = 20

# Decay of inhibitory syanapses
decay_n = lambda t: (t / 50) * np.exp(1 - (t / 50)) ; howfar_n = 100

# Input to cortex
input_to_cortex = lambda t : 7*np.random.normal(1, 3, size=(sizes[0],1))

# Weight randomization w <- (1 + nu)*w with nu gaussian process of mean 0
def randomized_w(weight, size):
    sign_w = np.sign(weight)
    sigma_w = np.random.normal(0, 1, size=size)
    w = abs(weight) + sigma_w * abs(weight)
    w[w < 0] = 0
    return sign_w * w


graph = tf.Graph()
with graph.as_default():
    
    # Populations
    nuclei = [Izhi_Nucleus(size, label=name, **parameters,
                           Iext=0 if name != 'CTX' else input_to_cortex)
              for name, size, parameters in zip(names, sizes, parameters)]

    # Connections between populations
    for i, N in enumerate(nuclei):
        for j, M in enumerate(nuclei):
            weight = connexion_matrix[i][j]/N.n
            delay = delay_matrix[i][j]
            if weight != 0:
                connect(N, M, randomized_w(weight, (M.n, N.n)), delay=delay,
                        decay=decay_p if weight > 0 else decay_n,
                        howfar=howfar_p if weight > 0 else howfar_n)
    
    # Building tensorflow graph for this model
    data = build_izhi(dt, nuclei)

# Simulate the model
vss, uss, Iss, firedss = simulate(T, dt, graph, nuclei, data)

# Raster plot of all the nuclei
for nucleus in nuclei:
    raster_plot(nucleus)

# Plot the activity of 1 neuron of TI
plot_neuron_by_idx(T, dt, {nuclei[-2] : [0, ]})
show()
```
