# HW3: Light Field Cameras

![Camera Array](http://graphics.stanford.edu/courses/cs348b-19-spring-content/article_images/6_.jpg)

Light field cameras (like the Lytro camera) allow users to capture a four-dimensional function of radiance (a light field), and then use it to synthesize new images with different focal planes, aperture sizes, or camera positions by sampling and reconstructing from this light field.

PBRT has four built-in camera models: simple orthographic and perspective cameras that can be augmented with a thin lens model, an environment camera for capturing a full 360 degree view of a single point, and a realistic camera that traces rays through a data-driven lens system before sending them out into the scene. First, we will add a new camera to this list - the light field camera - that captures the scene with a two dimensional grid of perspective cameras to generate a light field for the scene. Once we can generate new light fields, we will build a new tool within PBRT to resample our light fields and generate novel views of the captured scenes.

# Step 0: Setup

We will be providing starter code for this assignment, so the setup will be somewhat different than assignments 1 and 2. Rather than creating a new branch based off your local master branch, you will create a branch from the `assignment3` branch in our starter code repository: [https://github.com/stanford-gfx/cs348b-starter-code](https://github.com/stanford-gfx/cs348b-starter-code).

To achieve this, run the following commands from your repository:
   
    git remote add starter_code https://github.com/stanford-gfx/cs348b-starter-code.git
    git fetch starter_code
    git checkout -b assignment3 --track starter_code/assignment3
    git push -u origin assignment3
    
You should now be in the `assignment3` branch on your local repository with future commits set to go to your personal repository on GitHub.

Next, download the starter scene files for the assignment from [here](http://graphics.stanford.edu/courses/cs348b-20-spring-content/uploads/assignment3.zip) and extract them into the scenes/assignment3 directory. 

# Step 1: Background Reading

Levoy and Hanrahan introduced light fields into the graphics literature in 1996 with the SIGGRAPH paper [Light Field Rendering](https://graphics.stanford.edu/papers/light/light-lores-corrected.pdf).

Section 1 introduces light fields and should be familiar from lecture. The most important part of the paper conceptually is Section 2, which describes their representation for light fields. In particular, pay attention to their parameterization of the light field with intersecting two planes that rays intersect: the camera plane (u - v) and the focal plane (s - t). Section 3.1 describes how to generate a light field from rendered images, exactly our goal in Step 2; however, we will not be applying the paper's prefiltering step during light field generation. The final relevant section of the paper for the assignment is Section 5 on constructing a new image from the light field, which corresponds to our Step 3. Note how values are looked up in the lightfield starting with plane intersections and how they use interpolation of the nearest samples to resample the radiance.    

# Step 2: Extend PBRT to Generate New Light Fields

As a first step, we will be adding a new camera type to PBRT: the `LightfieldCamera`. By leveraging PBRT's flexible interfaces, we will be able to place this new camera in any `.pbrt` scene file. Rather than capturing a single view of the scene like a normal camera, our `LightfieldCamera` will capture a full light field that you will use to synthesize new views in Step 3.

For this assignment, we will be representing the light field as a 2D array of equally-spaced pinhole cameras with identical resolution and field of view. For the rest of the assignment, we will refer to these sub-cameras as *data cameras* - we capture the light field with a 2D array of these data cameras and save the view from each data camera to a shared film plane in a 2D grid. A scaled down example of the light field image you will be generating is shown below:

![scaled lightfield](http://graphics.stanford.edu/courses/cs348b-20-spring-content/article_images/6_16.jpg)

The skeleton of a `LightfieldCamera` class that inherits from the PBRT `Camera` base class has already been implemented for you in `src/cameras/lightfield.h` and `src/cameras/lightfield.cpp`. The starter code takes care of reading parameters for your camera from the `.pbrt` file. Specifically, the following parameters are read in and passed to the `LightfieldCamera` constructor for your use:

- `fov` - The field of view of the individual data cameras.
- `camerasperdim` - The number of data cameras per column and row in the light field capture array
- `cameragridbounds` - The bounds (in meters) of the light field camera grid in world space.

Putting this together, the following `LightfieldCamera` could be specified in the scene:

    Camera "lightfield" "float fov" [ 50 ] "integer camerasperdim" [16 16] 
           "float cameragridbounds [-0.6 0.6 -0.6 0.6]"

This will create a light field camera with 256 data cameras (each with a 50 degree field of view) arranged in a 16x16 grid. The cameras are equally spaced within an X,Y bounding box that goes from [-0.6] to [0.6] in both dimensions (the parameters to `cameragridbounds` are [minX maxX minY maxY]).

Your implementation of `LightfieldCamera` will need to fill out the `LightfieldCamera` constructor and `LightfieldCamera::GenerateRayDifferential`. Be sure to read the starter code and the provided comments, which provide references to sections in the PBR book that document the classes you will be using. 

The constructor should initialize and save any state required by ray generation in member variables within the `LightfieldCamera` class. Specifically, the constructor should create one PBRT `PerspectiveCamera` for each data camera in the grid and store those cameras within some container in the `LightfieldCamera`. Each `PerspectiveCamera` will require a pointer to a `CameraToWorld` transform, which must remain valid for the duration of the `PerspectiveCamera`'s lifetime. One easy way to ensure this is to first construct the `Transform` for the `PerspectiveCamera`, and then push that transform into a container within your `LightfieldCamera`, at which point you can use a pointer to the `Transform`'s position in the container.  Although there are multiple strategies, we recommend constructing the `PerspectiveCamera` with a transform from the individual data camera's camera space to the global `LightfieldCamera` camera space.

`LightfieldCamera::GenerateRayDifferential` is responsible for generating rays into the scene given a `CameraSample`. Based on the film position of the sample, your implementation should choose the correct `PerspectiveCamera` data camera and simply delegate the `GenerateRayDifferential` call to it, while not forgetting about any necessary transforms.

Once you have finished implementing the `LightfieldCamera`, render `scenes/assignment3/book-small.pbrt`, located in the step2 folder. This light field is too small to be useful but should render very quickly for easy testing. A correct implementation should produce the following light field:

![Smallfield](http://graphics.stanford.edu/courses/cs348b-20-spring-content/article_images/6_14.jpg)

After working out any bugs in your code, render the light field that will be used for step 3: `scenes/assignment3/book.pbrt`. Generating light fields is computationally expensive, so this may take a couple hours on your machine. The end result will be saved to `book.exr` and should look like a higher resolution version of the image at the beginning of this section.

**Implementation Tips**:

- Be consistent in how your implementation maps from film positions on the shared film plane to data cameras. The reference solution maps the top left camera (when viewed from the back of the grid along the camera axis), to the top left position on the film plane.
- Your implementation may assume that the number of data cameras in each dimension will perfectly divide the shared film plane into equally sized squares.
- You may use any container you would like for storing the `PerspectiveCamera`s and `Transform`s, but the easiest option is to use two `std::deque`s, which allow indexing like a `std::vector`, but also ensure that adding new elements will not invalidate earlier pointers to elements in the container.
- The `PerspectiveCamera` class requires an `AnimatedTransform` to be passed to its constructor to support moving the camera during the shot. We will not be supporting this feature, so simply construct an `AnimatedTransform` by passing in the same `Transform` pointer for both `start` and `end`.

# Step 3: Synthesize a novel image from a light field

In this section, we will be building a tool to synthesize new views from the light field you generated in step 2. The new view onto the scene will be defined by the concept of a "virtual" camera, which is placed relative to the light field camera array. Additionally, to allow customizing depth of field and focus, the virtual camera will be modeled with a thin lens that allows defining lens radius and focal distance for the new view.

Mechanically, we will synthesize new views by mimicking the behavior of a standard Monte Carlo ray tracer: given a set number of samples per pixel, rays will be launched from the virtual camera to build a Monte Carlo estimate of the irradiance arriving at each pixel in the virtual camera's output film. Rather than intersecting these rays with a geometric representation of the scene, the radiance values of these rays will be looked up in the generated light field from step 2. This strategy will allow us to treat individual rays as simply as possible, while still providing the ability to generate new views with varying camera orientations and lens parameters. The diagram below illustrates the setup:

![diagram](http://graphics.stanford.edu/courses/cs348b-20-spring-content/article_images/6_13.jpg)

The starter code for this step is in `src/tools/lightfield.cpp`, which is compiled into a standalone executable named `lightfield` in your PBRT build directory. The `lightfield` executable has argument parsing functionality already implemented for you, with arguments for both defining the parameters of the input light field (these must match the parameters used to generate the light field), as well as parameters for specifying the virtual camera. To familiarize yourself with these arguments, run `lightfield` with no arguments to see all the parameters and their descriptions (defined in the `usage()` function). In the code, these parameters and their default values are stored in the `LightfieldParameters` struct, defined at the top of the file.

Synthesizing a new view from the light field begins in the `Render` function, which defines a render loop modeled after PBRT's `Integrator::Render`. Your first step should be to read through this function and understand its implementation. At a high level, it first calls `LightfieldManager::MakeVirtualCamera` to create the virtual camera that will generate rays through the light field. Then it loops over every pixel and sample in the output image, generating rays from the virtual camera and calling `LightfieldManager::ComputeRadiance` to calculate the radiance of each ray.

The `LightfieldManager` class currently only contains code for reading the light field off the disk: the image itself and its resolution are saved in the `lightfieldImage` and `lightfieldResolution` member variables respectively. You will need to implement `LightfieldManager::MakeVirtualCamera` and `LightfieldManager::ComputeRadiance`, which will also require extending the `LightfieldManager` constructor to derive necessary parameters and transforms from the `LightfieldParameters` for these two functions.

`LightfieldManager::MakeVirtualCamera` should simply construct a PBRT `PerspectiveCamera` with settings corresponding to the desired output camera. This will allow `Render` to generate random rays corresponding to the specified camera position and lens configuration.

`LightfieldManager::ComputeRadiance` takes in a `Ray` and calculates the radiance (represented as a `RGBSpectrum`) along the ray. This function will be the bulk of your implementation. Our saved light field from step 2 is only a finite discrete sampling of the 4D light field. Since the virtual camera will generate rays through a continuous space, we need some strategy for *interpolating* our discrete light field to compute the radiance along a specific ray. There are many possible strategies for sampling the radiance from the saved light field, but for this assignment you will implement a relatively simple strategy:

 - First, find where the ray intersects the data camera plane. Recall that in terms of the captured light field, the data camera plane is located at z = 0 with its normal facing down the z axis.
 - Next, find where the ray intersects the virtual camera's focal plane. The focal plane is defined relative to the orientation of the virtual camera, and its distance from the virtual camera is defined by the `focaldistance` parameter.
 - Find the 4 nearest data cameras to the intersection between the ray and the data camera plane.
 - For each of those data cameras, define a new ray from the origin of the data camera to the intersection of the original ray with the focal plane. We will call these rays "data camera rays".
 - For each data camera ray, apply the inverse of the standard camera transforms to find the corresponding point on the film plane.
 - For each data camera, use bilinear interpolation of the stored pixel values to retrieve the correct radiance value corresponding to the point on the film plane. Be sure to handle boundary conditions (don't read off the edge of the shared film, and make sure you're not interpolating into a neighboring camera's pixels).
 - Given a radiance value from each of the data cameras, use bilinear interpolation between the data cameras (based on their relative distance from the data camera plane intersection point) to compute the final radiance value for the ray.

This algorithm is illustrated for both the the 2D and 3D cases in the below diagrams:

![Diagram](http://graphics.stanford.edu/courses/cs348b-19-spring-content/article_images/6_3.jpg)

Once you have correctly implemented this section, you should be able to run:

    lightfield --samplesperpixel 16 --focaldistance 6.8 --camerarot 5 -2.8 0 \\
               --camerapos 0.3 0.35 -0.8 --inputfov 50 --outputfov 40 --lensradius 0.3 \\
               --griddim -0.6 0.6 -0.6 0.6 --camsperdim 16 16 --outputdim 600 600 \\
               book.exr resolved.exr
            
`resolved.exr` should now contain an image very similar to: 

![this](http://graphics.stanford.edu/courses/cs348b-20-spring-content/article_images/6_17.jpg)

Note the noise in the reconstructed image. This is due to the non zero lens radius combined with too few samples per pixel: there simply are not enough samples to average for the correct defocus blur. We recommend debugging with a small number of samples, but for submission we require you to render a new view with at least 64 spp. 

For your submission, change the camera position, rotation and lens parameters to generate a new view of the book with the authors' names in focus. Please include some depth of field blur in your final result - do not simply set the lensradius to 0 to put the entire image in focus. 


**Implementation Tips**:

 - Note that the input light field is stored as a flat, row-major, array of `RGBSpectrum`. `RGBSpectrum` wraps up RGB values used to store radiance/irradiance/sensor response; addition of RGBSpectrums and multiplication by a `Float` are defined, so you never need to actually grab the individual channels for bilinear interpolation.
 - Similarly to step 2, `LightFieldManager::MakeVirtualCamera` will require a pointer to a `Transform` to pass to the `PerspectiveCamera`. The easiest option is to store the transform as a member variable of `LightfieldManager` in the `LightfieldManager` constructor.
 - Precompute as much as possible outside the `ComputeRadiance` function so you can synthesize new views quickly.


# Step 4: Short Discussions

- What happens to the images you generate if your lightfield is undersampled (the number of cameras per dimension is small)? Why?
- What happens to your final images if you seek to fix the artifacts described in the previous question by using many cameras but with very low resolution (say a 128x128 grid of 64x64 resolution cameras)?
- How would you modify the approach in step 3 (and the implementation of `LightFieldCamera`) to support sampling the light field in time in addition to the current implementation that samples in space?


# Submission

Once you are done with the assignment, convert your light field from step 2 and your final image from step 3 to png files using the following command:

     imgtool convert --tonemap [imagename].exr [imagename].png
     
Save the final pngs in `scenes/assignment3/step2.png` and `scenes/assignment3/step3.png` respectively.

Next, create a `Submission.md` file in the root of your repository that contains the following information:

- Your answers to the short discussion questions in Step 4.
- A description of any troubles you ran into while implementing the assignment, and how you overcame or worked around them.
- An embedded copy of `step3.png` using markdown syntax as described in the assignment 2 handout.
- The `lightfield` command arguments you used to generate `step3.png` (camerapos, camerarot, etc...).

In total, your submission should include the following files:

- All your changes to `src/cameras/lightfield.h`, `src/cameras/lightfield.cpp` and `src/tools/lightfield.cpp` (as well as any other changes you made to PBRT).
- The images produced in steps 2 and 3 as png files.
- Your Submission.md file.

Ensure that all your changes have been committed and pushed to GitHub, then create a pull request from your assignment3 branch to *your* master branch. Add the TAs as reviews of the pull request and **DO NOT** merge your pull request. Keep your master branch unchanged as a basis for future assignments.

# Grading
- 0 - Little or no work was done.
- 1 - Significant effort was put into the assignment, but not everything works correctly, insufficient explanation given, or answers incorrect.
- 2 - Everything mostly works in step 2 & 3, and errors are explained, short answers sufficient, adequate discussion of troubles during implementation.
- 3 - Everything works, images for steps 2 & 3 go above and beyond minimum requirement, short answers are insightful and precise, detailed record of troubles during implementation.
