<div align="center">
	<img src="./logo-detech.png" alt="Detech logo" width="320" />
</div>

# Detech Algorithm

*The Detech web app for image pre-processing and machine learning.*

The app is composed by an API module written in Go and a Core Module written in Python. The API module contains all the URL endpoints that call the scripts in the Core Module.

We first perform a pre-processing of the images, i.e., file name changing, alignments, flips, registering. Then, for the machine learning  we use a Bidirectional Recurrent Neural Network for training and testing.

The frontend template was created using Vue.js, and makes the calls to the API endpoints via jQuery ajax methods.


## Modules

- Core Module: Image processing and Machine Learning backend written in Python.
- API Module: controllers that make calls to the core module functions via bash shell. This module is written in Go.


## API Module

### [main.go](./main.go)  
Go's main function, which contains [Iris](https://github.com/kataras/iris) web framework initialization.  

Mapped routes:  

**`app.Get("/", func(c context.Context)`**  
Registers the initial view template.  
 
**`app.Post("/upload-folder", func(c context.Context)`**  
Receives a local folder location and copies all the contained photos with the `detechPhoto` format in a new folder called `UPLOAD_FOLDER`. It also executes the [`core/create_workspace.sh`](./core/create_workspace.sh) command, which calls the [`core/detechToTags.py`](./core/detechToTags.py).


### [sorting.go](./sorting.go)  
Contains the implementation of our custom sorting function in Go. We create a `ByDetechTag` type, which is just an alias for the builtin `[]string` type. We implement `sort.Interface` - `Len`, `Less`, and `Swap` - on our type so we can use the sort package’s generic Sort function. `Len` and `Swap` will usually be similar across types and `Less` will hold the actual custom sorting logic.

**`func (s ByDetechTag) Len() int`**   
Returns the length of an array of strings.

**`func (s ByDetechTag) Swap(i, j int)`**  
Swaps the position in array of two strings.

**`func (s ByDetechTag) Less(i, j int) bool`**  
Compares alphabetically two string in position i and j. Both strings need to have the 'detech' tag in their name. 


### [api.go](./api.go) 
Extends API by defining routes to be called by the frontend.

Mapped routes:

**`app.Get("/i/delete_all", func(c context.Context)`**  
Deletes `UPLOAD_FOLDER`, `WORKSPACE_FOLDER`, `REGISTRATION_FOLDER`, `CROPPED_FOLDER` and `DIFFS_FOLDER`, and recreates them empty. 


**`app.Get("/i/names", func(c context.Context)`**  
Reads all the files in `WORKSPACE_FOLDER`, sorts them by file name and prints them.


**`app.Get("/i/{name}", func(c context.Context)`**  
Downloads the file with the specified name in the URL, from the `WORKSPACE_FOLDER` folder.


**`app.Get("/i/registration", func(c context.Context)`**  
Executes the bash script `core/register_workspace.sh` and passes two arguments: `WORKSPACE_FOLDER` as input, and `REGISTRATION_FOLDER` as output.


**`app.Get("/i/reg/{name}", func(c context.Context)`**  
Downloads the file with the specified name in the URL, from the `REGISTRATION_FOLDER` folder.


**`app.Get("/i/mean", func(c context.Context)`**  
Executes the bash script `core/get_mean_image.sh` and passes the `REGISTRATION_FOLDER` as argument. The method returns a JSON object with the name of the last image from the output.


**`app.Get("/i/crop", func(c context.Context)`**  
Executes the bash script `core/cropping_workspace.sh` and passes three parameters: `REGISTRATION_FOLDER`, `CROPPED_FOLDER`, and pads. Returns a JSON object with the output of the script.


**`app.Get("/i/crop/{name}", func(c context.Context)`**  
Downloads the file with the specified name in the URL, from the `CROPPED_FOLDER` folder.


**`app.Get("/i/diffs", func(c context.Context)`**  
Executes the bash script `core/apply_diff.sh` and passes three parameters `CROPPED_FOLDER`, `DIFFS_FOLDER`, and params. Then, it reads all the files from `DIFFS_FOLDER` and sorts them by file name. Returns the list of file names sorted.


**`app.Get("/i/diffs/names", func(c context.Context)`**  
Reads all the files in `DIFFS_FOLDER`, sorts them by file name and prints them.


**`app.Get("/i/diffs/{name}", func(c context.Context)`**  
Downloads the file with the specified name in the URL, from the `DIFFS_FOLDER` folder.


**`app.Get("/i/diffs/clean", func(c context.Context)`**  
Deletes the `DIFFS_FOLDER` and recreates it.


## Core Module

### Bash files
These files execute the Python scripts.

* [`create_workspace.sh`](./core/create_workspace.sh)  
	Executes the Python script `core/detechToTags.py` with two parameters: `--input ` and `--output`.
	
* [`apply_diff.sh`](./core/apply_diff.sh)  
	Executes the Python script `core/apply_diffs.py` with three parameters: `--input`, `--output`, and `--params`.

* [`cropping_workspace.sh`](./core/cropping_workspace.sh)  
	Executes the Python script `core/cutting.py` with three parameters: `--input`, `--output`, and `--pads`.

* [`get_mean_image.sh`](./core/get_mean_image.sh)  
	Executes the Python script `core/mean_image.py` with one parameter: `--input`.

* [`register_workspace.sh`](./core/register_workspace.sh)  
	Executes the Python script `core/registrationOfImages.py` with two parameters: `--input ` and `--output`.


### Python primary scripts

* [`detechToTags.py`](./core/detechToTags.py)  
	Renames `detech` nomenclature files with a specific format for making them easier to process. Copies the renamed files in the `output_path`, which is the `WORKSPACE_FOLDER` as specified in the `main.go`file.
	
	The function receives two parameters: 
	* input_path: `UPLOAD_FOLDER`.
	* output_path: `WORKSPACE_FOLDER`.
	
	
	```python
	def detech2tags(input_path, output_path):
	```
	
	**Each photo is taken with the camera gun device at 8cm, 10cm or 12cm of distance to skin.**
	
	```python
	dist = ['8', '10', '12']
	```
	
	```python
	hand = 'a'
	num = (filesp[j][1]-filesp[0][1]).seconds
	w = 0.
	if num!=0:
		w = num/totalSeconds

	fname = 'img'+str(filesp[j][0])+'_'+hand+'_'+dist[m]+'_'+str(w).replace('.', 'd') + '.jpg'
    ```
    
    Here, we specify the hand type (**'a' for the affected hand, and 'b' for the unaffected one**). Then, we calculate a **weight** for each photo based on the seconds transcurred from the **beginning** to the capture of the photo. 
    
    Finally, we rename each file to obtain a name in the form: `img101_pt3_a_8_0d638547712242.jpg`
	
* [`apply_diffs.py`](./core/apply_diffs.py)  
	Substracts photos to find differences. This allows us to get the evolution of the dermal ulcer.

	The script receives three arguments:
	* --input: the input folder, `CROPPED_FOLDER`.
	* --output: the output folder, `DIFFS_FOLDER`.
	* --params: the parameter for difference mode passed by user as URL parameter. By default it passes the `side` and `type` values defined in the Vue `data` object from the [templates/index.html](./templates/index.html) file. 

	The difference extraction is performed by the `folderDiffsExtract` function, which is defined in the [`diffs_and_threshold.py`](./core/diffs_and_threshold.py) file.

* [`cutting.py`](./core/cutting.py)  
	Crops photos using the `pads` parameter. The processed images are saved in the output folder: `CROPPED_FOLDER`.
	
	The script receives three arguments:
	* --input: the input folder, `REGISTRATION_FOLDER `.
	* --output: the output folder, `CROPPED_FOLDER `.
	* --pads: the pads for bounding passed by user as a comma-separated URL parameter.

* [`mean_image.py`](./core/mean_image.py)  
	Calls the function `calculateTheMostNearImageToMeanImage` from the file [images_operations.py](./core/images_operations.py) and passes the `inputFolder` as parameter. Then prints the image received from this function.

* [`registrationOfImages.py`](./core/registrationOfImages.py)  
	Applies a rigid registration to all the photos in the input folder. This registration is performed in relation with the closest image to the mean image. 
	
	For each file in the input folder executes the `RigidRegistration(...)` function from the [core/registro1.py](./core/registro1.py) script:
	
	```python
	protoImg = reg.RigidRegistration(inputFolder+nameOfMediumImage, inputFolder+f, 'correlation', 'grad-desc')
	```


### Python secondary scripts

* [`diffs_and_threshold.py`](./core/diffs_and_threshold.py)  
Contains the function `folderDiffsExtract(...)`, which iterates over the files in the input folder and separates the images based on whether they belong to `a` or `b` hand. Then calculates the difference between the `a` image and the `b` image.  
In case `tsub == 'ab'` we substract image 2 from image 1. In the other case (`tsub == 'ba'`) we substract image 1 from image 2.

	```python
	if tsub[0] == 'ab':
	    diff = ImageChops.difference(img1, img2)
	    finalWeight = im1Weight - im2Weight
	else: # or if tsub == 'ba'
	    diff = ImageChops.difference(img2, img1)
	    finalWeight = im2Weight - im1Weight
	```
	
	We then assign a formatted file name, including the distance of the camera (8cm, 10cm or 12cm), and save the diff file in the output folder.

	```python
	name = 'img' + str(j) + '_d8_' + str(finalWeight).replace('.', 'd') + '.jpg'
	diff.save(outputFolder + name)
	```

	If the `tsub` parameter is equal to `'t'` the images difference are calculated taking two images at a time from the same hand (whether 'a' or 'b').

* [`images_operations.py`](./core/images_operations.py)  
Contains functions that process images.  

	* `checkIfIsAorB(file)`  
		Returns `a` or `b` depending on the file name.
		
	* `calculateMediumImageOfFolder(folderPath)`  
		Uses `numpy` library to calculate the medium image of a folder. 		This is done by converting each image to an array, summing them 		all and dividing by the number of images.
		
		In order to correctly calculate the sum and difference of the images, we first need to make the necessary flips and transformations.
		
		```python
		for f in files:
		    if checkIfIsAorB(f) == 'a':
		        readyImages.append(np.asarray(Image.open(folderPath + f)))
		    else:
		        img = ImageOps.flip(Image.open(folderPath + f))
		        img = ImageOps.mirror(img)
		        readyImages.append(np.asarray(img))
       ```

	* `calculateTheMostNearImageToMeanImage(folder)`  
		Calculates the error of each image in the folder in comparison 		with the mean image. It returns the image processed as array, as 		well as the original file. 

	* `flipAllBTypeImagesFromFolder(folder, typeFlip)`  
		Flips all `b` images from the folder received as parameter. 		Dependding on the `typeFlip` parameter, it can perform a flip, a 		mirror or both operations.	
		
* [`registro1.py`](./core/registro1.py)  
Contains the function for Rigid Registration. This function receives parameters for customizing the behavior of the registration algorithm. 

* [`registro2.py`](./core/registro2.py)  
Contains the function for applying a B-Spline Registration. This functions receives two parameters:  
	* `imagen1`: the fixed image.
	* `imagen2`: the moving image.

* [`detech_data.py`](./core/predictor/model/detech_data.py)  
Contains a `DetechDataset` class with its defined functions. This class works with the [`testOut`](./core/predictor/model/testOut) folder, which contains the images in a numpy format.

* [`bidir_rnn.py`](./core/predictor/model/bidir_rnn.py)
Contains the definition of a Bidirectional Recurrent Neural Network algorithm used for training and testing using data from the `DetechDataset` object.


## Dependencies

* Python 2.7
* [Go tools](https://golang.org/doc/install)


## Installation

Run the following script for downloading and installing packages and dependencies.

```
$ go get github.com/bregydoc/detech/detechAlgorithm
```

Get into the downloaded folder and run the `main.go` file.

```
cd detechAlgorithm
go run main.go
```

## Notes
* In all the scripts, we consider `'a'` as the affected hand, and `'b'` as the unaffected one.
* Each photo is taken with the camera gun device at 8cm, 10cm or 12cm of distance to skin.
 

## License


## Authors
