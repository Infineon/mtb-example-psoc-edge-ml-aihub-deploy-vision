[Click here](../README.md) to view the README.

## Design and implementation

The design of this application is minimalistic to get started with code examples on PSOC&trade; Edge MCU devices. All PSOC&trade; Edge E84 MCU applications have a dual-CPU three-project structure to develop code for the CM33 and CM55 cores. The CM33 core has two separate projects for the secure processing environment (SPE) and non-secure processing environment (NSPE). A project folder consists of various subfolders, each denoting a specific aspect of the project. The three project folders are as follows:

**Table 1. Application projects**

Project | Description
--------|------------------------
*proj_cm33_s* | Project for CM33 secure processing environment (SPE)
*proj_cm33_ns* | Project for CM33 non-secure processing environment (NSPE)
*proj_cm55* | CM55 project

<br>

In this code example, at device reset, the secure boot process starts from the ROM boot with the secure enclave (SE) as the root of trust (RoT). From the secure enclave, the boot flow is passed on to the system CPU subsystem where the secure CM33 application starts. After all necessary secure configurations, the flow is passed on to the non-secure CM33 application. Resource initialization for this example is performed by this CM33 non-secure project. It configures the system clocks, pins, clock to peripheral connections, and other platform resources. It then enables the CM55 core using the `Cy_SysEnableCM55()` function and puts itself (CM33) into DeepSleep mode. The main firmware executes on the CM55 core.

In the CM33 non-secure application, the clocks and system resources are initialized by the BSP initialization function. The retarget-io middleware is configured to use the debug UART. The debug UART prints a message (as shown in [Terminal output on program startup](../images/terminal-output.png)) on the terminal emulator, the onboard KitProg3 acts the USB-UART bridge to create the virtual COM port. The User LED blinks every 1 second.

Once CM55 is enabled it configures system clocks, pins, clock to peripheral connections, and other platform resources.

The code example is a real-time human segmentation detection demo running on a PSOC&trade; Edge MCU, the CM55 application drives the HBVCAM OV7675 0.3MP MINI Camera for real-time image capture, an object detection model for human segmentation recognition, and a graphics subsystem for displaying results on Waveshare 4.3 inch Raspberry Pi DSI 800 x 480 pixel display. The graphics subsystem of PSOC&trade; Edge MCU houses an independent 2.5D graphics processing unit (GPU), a display controller (DC), and a MIPI DSI host controller with MIPI D-PHY physical layer interface. The example is built using a FreeRTOS to manage three concurrent tasks: USB webcam handling, object detection inference, and graphics rendering.

By default, the CM55 application places the model weights in the SoCMEM for best performance. The application *Makefile* then uses `CY_ML_MODEL_MEM` to set the location for the model weights.

By default, the code example uses the person segmentation model. However, you can replace with different models and deploy on PSOC&trade; devices. To use different models, see [DEEPCRAFT&trade; - Model Zoo for PSOC&trade;](https://github.com/Infineon/deepcraft-model-zoo-for-psoc) and for additional model details, see [DEEPCRAFT&trade; AI Hub](https://deepcraft.infineon.com/).

Following are the key parts in this code example:

**Application code:** The `main()` function in *proj_cm55* > *main.c* file for the initialization of board support package (BSP), retarget-io for debug UART communication, and the creation of the three FreeRTOS tasks. In the following, each task is described with its design and implementation details.

- Webcam task (`cm55_usb_webcam_task`)

  - The `cm55_usb_webcam_task`, defined in *proj_cm55* > *usb_camera_task.c* file, initializes the USB host stack and manages communication with USB cameras. It registers the video device module with the emUSB-Host stack, detects connected devices, configures video settings, and processes video frames for subsequent inference

  - The task initializes the USB host stack with `USBH_Init()` and registers the video device module using `USBH_VIDEO_Init()`. It also sets up a notification hook (`_cbOnAddRemoveDevice`) to handle dynamic device connection/disconnection events

  - Two additional threads (`USBH_Task` and `USBH_ISRTask`) are created to handle USB host operations and interrupts. A mailbox queue (`_VIDEO_MailBox`) is initialized to receive device notifications, triggering `_OnDevReady()` to process the device index, enabling non-blocking operation

  - Upon detecting a video interface, the task displays device information and configures video control settings. It then reads multiple video frames, storing them in buffers for processing by the inference task

  <br>

- Inference task (`cm55_inference_task`)

  - The `cm55_inference_task`, defined in *proj_cm55* > *inference_task.c* file initializes and runs an object detection model to identify human segmentation from camera-captured images. It handles image preprocessing, model inference, and postprocessing, scaling input images to match the model’s requirements and synchronizing results with the graphics task via a semaphore (`model_semaphore`).

  - The task initializes the object detection model using `ml_pipeline_model_init()`

  - Images are preprocessed using `ml_pipeline_preprocess()` to convert camera frames to the model’s required format

  - The task executes model inference with `ml_pipeline_inference()` and applies postprocessing (`ml_pipeline_post_process()`) to filter detections based on configurable parameters

 <br>

- Graphics task (`cm55_ns_gfx_task`)

  - The `cm55_ns_gfx_task`, defined in *proj_cm55* > *lcd_task.c* file, initializes the graphics subsystem and LCD display driver (e.g., Waveshare 4.3-inch display) to render camera images and object detection results. The task initializes the graphics subsystem with `Cy_GFXSS_Init()` and configures interrupts for the display controller. The I2C interface is set up for display communication

  - A transformation matrix (`matrix`) scales and centers camera images on the display. The task updates the display with inference results (`update_box_data()`) and performance metrics (`Inference_time`) upon receiving a semaphore signal (`model_semaphore`)

  - Double buffering and `vg_lite_flush()` minimize rendering artifacts, while frame rate calculations monitor display performance.

  <br>