Shogun NeuralNetworks vs OpenCV Neural Network.

```CPP
#include <shogun/base/init.h>
#include <shogun/lib/SGVector.h>
#include <shogun/lib/SGMatrix.h>
#include <shogun/features/DenseFeatures.h>
#include <shogun/labels/MulticlassLabels.h>
#include <shogun/lib/DynamicObjectArray.h>
#include <shogun/neuralnets/NeuralNetwork.h>
#include <shogun/neuralnets/NeuralInputLayer.h>
#include <shogun/neuralnets/NeuralLogisticLayer.h>
#include <shogun/lib/OpenCV/CV2FeaturesFactory.h>
#include <shogun/lib/OpenCV/CV2SGMatrixFactory.h>

#include <iostream>

// opencv includes.
#include<opencv2/ml/ml.hpp>
#include<opencv2/core/core.hpp>
#include<opencv2/highgui/highgui.hpp>

using namespace shogun;
using namespace std;
using namespace cv;

```

```CPP

int main()
{
    init_shogun_with_defaults();
    CvMLData mlData;
    mlData.read_csv("car.data");
    const CvMat* temp = mlData.get_values();
    int numfeatures = temp->cols-1;
    mlData.set_response_idx(numfeatures);
    CvTrainTestSplit spl((float)0.5);
    mlData.set_train_test_split(&spl);  
    const CvMat* traindata_idx = mlData.get_train_sample_idx();
    const CvMat* testdata_idx = mlData.get_test_sample_idx();
```
```CPP
    
    Mat mytraindataidx(traindata_idx);
    Mat mytestdataidx(testdata_idx);
    Mat all_Data(temp);
    Mat all_responses = mlData.get_responses();
    Mat traindata(mytraindataidx.cols,numfeatures,CV_32F);
    Mat trainresponse(mytraindataidx.cols,1,CV_32S);
    Mat testdata(mytestdataidx.cols,numfeatures,CV_32F);
    Mat testresponse(mytestdataidx.cols,1,CV_32S);
```
```CPP
    for (int h=0; h<all_responses.rows; h++)
    {
        if (all_responses.at<float>(h) == 4 )
            all_responses.at<float>(h)=0;

        else if (all_responses.at<float>(h) == 10)
            all_responses.at<float>(h)=1;

        else if (all_responses.at<float>(h) == 11)
            all_responses.at<float>(h)=2;

        else all_responses.at<float>(h)=3;
    }
```
```CPP
    for(int i=0; i<mytraindataidx.cols; i++)
    {
        trainresponse.at<int>(i)=all_responses.at<float>(mytraindataidx.at<int>(i));    
        for(int j=0; j<=numfeatures; j++)
        {
            traindata.at<float>(i, j)=all_Data.at<float>(mytraindataidx.at<int>(i), j);
        }
    }

    for(int i=0; i<mytestdataidx.cols; i++)
    {
        testresponse.at<int>(i)=all_responses.at<float>(mytestdataidx.at<int>(i));
        for(int j=0; j<=numfeatures; j++)
        {
            testdata.at<float>(i, j)=all_Data.at<float>(mytestdataidx.at<int>(i), j);
        }   
    }

```


```CPP
    SGMatrix<float64_t> shogun_traindata = CV2SGMatrixFactory::getSGMatrix<float64_t>(traindata, CV2SG_MANUAL);
    SGMatrix<float64_t>::transpose_matrix(shogun_traindata.matrix, shogun_traindata.num_rows, shogun_traindata.num_cols);
    CDenseFeatures<float64_t>* shogun_trainfeatures = new CDenseFeatures<float64_t>(shogun_traindata);
```
```CPP
    CDenseFeatures<float64_t>* shogun_dense_response = CV2FeaturesFactory::getDenseFeatures<float64_t>(trainresponse, CV2SG_MANUAL);
    SGVector<float64_t> shogun_vector_response = shogun_dense_response->get_feature_vector(0);
    CMulticlassLabels* labels = new CMulticlassLabels(shogun_vector_response);
```

```CPP
    SGMatrix<float64_t> shogun_testdata = CV2SGMatrixFactory::getSGMatrix<float64_t>(testdata, CV2SG_MANUAL);
    SGMatrix<float64_t>::transpose_matrix(shogun_testdata.matrix, shogun_testdata.num_rows, shogun_testdata.num_cols);
    CDenseFeatures<float64_t>* testfeatures = new CDenseFeatures<float64_t>(shogun_testdata);
```
___

To use NN in shogun following things are needed to be done

* Prepare a CDynamicObjectArray of CNeuralLayer-based objects that specify the type of layers used in the network. The array must contain at least one input layer. The last layer in the array is treated as the output layer. Also note that forward propagation is performed in the order at which the layers appear in the array. So if layer j takes its input from layer i then i must be less than j.

* Specify how the layers are connected together. This can be done using either connect() or quick_connect().

* Call initialize()

* Specify the training parameters if needed

* Train set_labels() and train()

* If needed, the network with the learned parameters can be stored on disk using save_serializable() (loaded using load_serializable())

* Apply the network using apply()

___

* Lets start with the first step.

We will be preparing aCDynamicObjectArray. It creates an array that can be used like a list or an array.
We then append information related to number of neurons per layer in there respective order.

Here I have created a 3 layered network. The input layer consists of 6 neurons which is equal to number of features.
The hidden layer has 10 neurons and similarly the output layer has 4 neurons which is equal to the number of classes.

```CPP
    CDynamicObjectArray* layers = new CDynamicObjectArray();
    layers->append_element(new CNeuralInputLayer(6));
    layers->append_element(new CNeuralLogisticLayer(10)); 
    layers->append_element(new CNeuralLogisticLayer(4));
```
___
* Here we have to make a connection between the three layers that we formed above. To connect each neuron of one layer to each one of the layer suceeding it, we can directly use ```quick_connect()```. However If particular connections are to be made separately, we may have to use ```connect()```.  

```CPP
    CNeuralNetwork* network = new CNeuralNetwork(layers);
    network->quick_connect();
```
___
* Initialize the network. The input is nothing but the standard deviation of the gaussian which is used to randomly initialize the parameters. We chose 0.1 here.

```CPP
    network->initialize(0.1);
```
___
* specify the training parameters if needed. 

```CPP
    network->epsilon = 1e-8;
```
___
* set labels and train!

```CPP
    network->set_labels(labels);
    network->train(shogun_trainfeatures);
```
___
* test it!

```CPP
    CMulticlassLabels* predictions = network->apply_multiclass(testfeatures);
    int32_t k=0;
    for (int32_t i=0; i<mytraindataidx.cols; i++ )
    {
        if (predictions->get_label(i)==testresponse.at<int>(i))
        ++k;
    }
    cout<<100.0*k/(mytraindataidx.cols)<<endl;
    return 0;
}
```

Output!
```CPP
    81.1343

```
Our accuracy for performing Multiclass classification using Shogun's NN is 81.13% on this dataset