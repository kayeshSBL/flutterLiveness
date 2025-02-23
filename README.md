# flutterLiveness


Hello everyone! In this video, we’ll build a Face Liveness Detection App using Flutter and Google ML Kit. This app detects if a real person is in front of the camera by analyzing facial movements like head rotation, smiling, and eye blinking. Let’s dive into the code and see how it works!"

import 'dart:convert';
import 'dart:io';
import 'dart:math';
import 'package:flutter/foundation.dart';
import 'package:flutter/material.dart';
import 'package:camera/camera.dart';
import 'package:flutter/services.dart';
import 'package:google_ml_kit/google_ml_kit.dart';
import 'dart:io' as i;
import 'dart:math' as Math;
 
class FaceLivenessDetection extends StatefulWidget {

  @override
  _FaceLivenessDetectionState createState() => _FaceLivenessDetectionState();
}

class _FaceLivenessDetectionState extends State<FaceLivenessDetection>  with WidgetsBindingObserver, TickerProviderStateMixin {
  CameraController? _cameraController;
  late FaceDetector _faceDetector;
  bool _isFaceInFrame = false;
  bool _isFaceLeft = false;
  bool _isFaceRight = false;
  bool _isEyeOpen = false;
  bool _isNoFace = false;
  bool _isMultiFace = false;
  bool _isCaptured = false;
  bool _isSmiled = false;
  bool _isFaceReadyForPhoto = false;

  var frontCamera;


  FaceMeshDetector _faceMeshDetector = GoogleMlKit.vision.faceMeshDetector();
  List<double>? _firstPersonEmbedding;
  bool _isDifferentPerson = false;




  XFile? _capturedImage;
  List<String> _successfulSteps = [];

  final FaceDetectorOptions options = FaceDetectorOptions(
    performanceMode: Platform.isAndroid ? FaceDetectorMode.fast : FaceDetectorMode.accurate,
    enableClassification: true, // Enable eye open probability detection
    enableLandmarks: true, // Landmarks are not needed
    enableTracking: true, // Disable face tracking
  );





  final orientations = {
    DeviceOrientation.portraitUp: 0,
    DeviceOrientation.landscapeLeft: 90,
    DeviceOrientation.portraitDown: 180,
    DeviceOrientation.landscapeRight: 270,
  };




  @override
  void initState() {
    super.initState();
    WidgetsBinding.instance.addObserver(this);
    _initializeCamera();
    //_faceDetector = GoogleMlKit.vision.faceDetector();
    _faceDetector = FaceDetector(options: options);

  }

  Future<void> _initializeCamera() async {
    final cameras = await availableCameras();
    final frontCameras = cameras.firstWhere(
          (camera) => camera.lensDirection == CameraLensDirection.front,
    );

    setState(() {
      frontCamera = frontCameras;
    });

    _cameraController = CameraController(
      frontCamera,
      ResolutionPreset.medium,
      imageFormatGroup: Platform.isAndroid
          ? ImageFormatGroup.nv21
          : ImageFormatGroup.bgra8888,
    );
    await _cameraController!.initialize();
    if (!mounted) return;
    setState(() {});
    _cameraController!.startImageStream((CameraImage img) {
      _processCameraImage(img);
    });
  }

  Future<void> _processCameraImage(CameraImage img) async {
    try {
      final inputImage = _getInputImageFromCameraImage(img);
      if (inputImage == null) return;

      final List<Face> faces = await _faceDetector.processImage(inputImage);


    if (faces.length > 1) {
      setState(() {
          _isMultiFace = true;
          _successfulSteps.clear();
          _resetFaceDetectionStatus();
        });
      }
      else if (faces.isEmpty) {
        setState(() {
           _isNoFace = true;
           _successfulSteps.clear();
           _resetFaceDetectionStatus();
        });
      }
      else if (faces.isNotEmpty) {
      _isMultiFace = false;
      _isNoFace = false;
        final Face face = faces.first;
      await _compareFaces(face); // Compare faces to detect different person

      if (_isDifferentPerson) {
        _duplicatePersonFaceDetect();
        return;
      }
        _handleFaceDetection(face);
      } else {
        _handleNoFaceDetected();
      }
    } catch (e) {
      print('Error processing camera image: $e');
    }
  }

  void _handleFaceDetection(Face face) {
    if(!_isCaptured) {


    final double? rotY = face.headEulerAngleY; // Head rotation angle
    final double leftEyeOpen = face.leftEyeOpenProbability ?? -1.0; // Left eye open probability
    final double rightEyeOpen = face.rightEyeOpenProbability ?? -1.0; // Right eye open probability
    final double smileProb = face.smilingProbability ?? -1.0; // Smiling probability

    print("Head angle: $rotY");
    print("Left eye open: $leftEyeOpen");
    print("Right eye open: $rightEyeOpen");
    print("Smiling probability: $smileProb");

    setState(() {
        _updateFaceInFrameStatus();
        _updateHeadRotationStatus(rotY);
        _updateSmilingStatus(smileProb);
        _updateEyeOpenStatus(leftEyeOpen, rightEyeOpen);
        _updateFaceInFrameForPhotoStatus(rotY,smileProb);
        if (_isFaceInFrame && _isFaceLeft && _isFaceRight && _isSmiled && _isFaceReadyForPhoto &&
             _isEyeOpen) {
          if (!_isCaptured) {
            _captureImage();
          }
        }
    });
    }
  }

  void _handleNoFaceDetected() {
    setState(() {
      if (_isFaceInFrame) {
        _resetFaceDetectionStatus();
      }
    });
  }

  void _duplicatePersonFaceDetect() {
    if (_isDifferentPerson) {
      _addSuccessfulStep('Different person Found');
      _resetFaceDetectionStatus();
    }
  }

  void _updateFaceInFrameStatus() {
    if (!_isFaceInFrame) {
      _isFaceInFrame = true;
      _addSuccessfulStep('Face in frame');
    }
  }


  void _updateFaceInFrameForPhotoStatus(double? rotY, double? smileProb) {
    if (_isFaceRight && _isFaceLeft && rotY != null && rotY>-2 && rotY<2 && smileProb! < 0.2) {
      _isFaceReadyForPhoto = true;
      _addSuccessfulStep('Face Ready For Photo');
    }else
      {
        _isFaceReadyForPhoto = false;
      }
  }



  void _updateHeadRotationStatus(double? rotY) {
    if (_isFaceInFrame && !_isFaceLeft && rotY != null && rotY < -7) {
      _isFaceLeft = true;
      _addSuccessfulStep('Face rotated left');
    }

    if (_isFaceLeft && !_isFaceRight && rotY != null && rotY > 7) {
      _isFaceRight = true;
      _addSuccessfulStep('Face rotated right');
    }
  }

  void _updateEyeOpenStatus(double leftEyeOpen, double rightEyeOpen) {
    if (_isFaceInFrame && _isFaceLeft && _isFaceRight && _isSmiled && !_isEyeOpen ) {
      if (leftEyeOpen > 0.3 && rightEyeOpen > 0.3) {
        _isEyeOpen = true;
        _addSuccessfulStep('Eyes Open');
      }
    }
  }

  void _updateSmilingStatus(double smileProb) {
    if (_isFaceInFrame && _isFaceLeft && _isFaceRight && !_isSmiled && smileProb > 0.3) {
      _isSmiled = true;
      _addSuccessfulStep('Smiling');
    }
  }

  void _resetFaceDetectionStatus() {
    _isFaceInFrame = false;
    _isFaceLeft = false;
    _isFaceRight = false;
    _isEyeOpen = false;
    _isNoFace = false;
    _isMultiFace = false;
    _isSmiled = false;
    _successfulSteps.clear();
  }

  void _addSuccessfulStep(String step) {
    if (!_successfulSteps.contains(step)) {
      _successfulSteps.add(step);
    }
  }


  InputImage? _getInputImageFromCameraImage(CameraImage image) {
    final sensorOrientation = frontCamera.sensorOrientation;
    InputImageRotation? rotation;
    if (Platform.isIOS) {
      rotation = InputImageRotationValue.fromRawValue(sensorOrientation);
    } else if (Platform.isAndroid) {
      var rotationCompensation =
      orientations[_cameraController!.value.deviceOrientation];
      if (rotationCompensation == null) return null;
      if (frontCamera.lensDirection == CameraLensDirection.front) {
// front-facing
        rotationCompensation = (sensorOrientation + rotationCompensation) % 360;
      } else {
// back-facing
        rotationCompensation =
            (sensorOrientation - rotationCompensation + 360) % 360;
      }
      rotation = InputImageRotationValue.fromRawValue(rotationCompensation!);
    }
    if (rotation == null) return null;
// get image format
    final format = InputImageFormatValue.fromRawValue(image.format.raw);
    if (format == null ||
        (Platform.isAndroid && format != InputImageFormat.nv21) ||
        (Platform.isIOS && format != InputImageFormat.bgra8888)) return null;
// since format is constraint to nv21 or bgra8888, both only have one plane
    if (image.planes.length != 1) return null;
    final plane = image.planes.first;
// compose InputImage using bytes
    return InputImage.fromBytes(
      bytes: plane.bytes,
      metadata: InputImageMetadata(size: Size(image.width.toDouble(), image.height.toDouble()),
        rotation: rotation, // used only in Android
        format: format, // used only in iOS
        bytesPerRow: plane.bytesPerRow, // used only in iOS
      ),
    );
  }


  Future<void> _captureImage() async {
    if (_cameraController!.value.isTakingPicture) return;
    try {
      final XFile file = await _cameraController!.takePicture();
      setState(() {
        _isCaptured = true;
        _capturedImage = file;
        final bytes = i.File(file.path).readAsBytesSync(); 
        _faceDetector.close();
      });
    } catch (e) {
      print(e);
    }
  }
 

  //  duplicate person detect
  Future<List<double>> _extractFaceEmbeddings(Face face) async {
    // Simulate face embeddings extraction (replace with actual face recognition model)
    return [
      face.boundingBox.left,
      face.boundingBox.top,
      face.boundingBox.right,
      face.boundingBox.bottom,
    ];
  }

  Future<void> _compareFaces(Face currentFace) async {
    final currentEmbedding = await _extractFaceEmbeddings(currentFace);

    if (_firstPersonEmbedding == null) {
      // First person detected
      setState(() {
        _firstPersonEmbedding = currentEmbedding;
      });
    } else {
      // Compare embeddings to check if it's the same person
      final double similarity = _calculateSimilarity(_firstPersonEmbedding!, currentEmbedding);
      setState(() {
        _isDifferentPerson = similarity < 0.8; // Threshold for similarity
      });
    }
  }

  double _calculateSimilarity(List<double> embedding1, List<double> embedding2) {
    // Calculate cosine similarity between two embeddings
    double dotProduct = 0.0;
    double norm1 = 0.0;
    double norm2 = 0.0;

    for (int i = 0; i < embedding1.length; i++) {
      dotProduct += embedding1[i] * embedding2[i];
      norm1 += embedding1[i] * embedding1[i];
      norm2 += embedding2[i] * embedding2[i];
    }

    return dotProduct / (Math.sqrt(norm1) * Math.sqrt(norm2));
  }

  //  end of duplicate person detect

  String _getCurrentDirection() {
    if (!_isFaceInFrame) {
      return 'Enter your face in the frame';
    } else if (_isNoFace) {
      return 'No Faces in Camera';
    }else if (_isMultiFace) {
      return 'Multi Faces in Camera';
    } else if (!_isFaceLeft) {
      return 'Rotate your face to the left (10° & 5 Sec)';
    } else if (!_isFaceRight) {
      return 'Rotate your face to the right (10° & 5 Sec)';
    }else if (!_isSmiled) {
      return 'Keep One Smile ';
    }else if (!_isEyeOpen) {
      return 'Open Your Eyes';
    }
    else if (!_isFaceReadyForPhoto) {
      return 'Ready For capture Photo, don\'t  laughing and keep strait your photo';
    }
    else {
      return 'Liveness detected! Image captured.';
    }
  }

  bool _isFaceInsideFrame(Rect boundingBox) {
    // Define the camera preview area (adjust as needed)
    const double previewWidth = 300;
    const double previewHeight = 300;

    // Check if the face bounding box is fully within the frame
    return boundingBox.left >= 0 &&
        boundingBox.top >= 0 &&
        boundingBox.right <= previewWidth &&
        boundingBox.bottom <= previewHeight;
  }



  @override
  void dispose() {
    _faceDetector.close();
    if (_cameraController != null) _cameraController!.dispose();
    WidgetsBinding.instance.removeObserver(this);
    _faceMeshDetector.close();
    super.dispose();
  }
  @override
  void didChangeAppLifecycleState(AppLifecycleState state) {
    final CameraController? cameraController = _cameraController;
    if (cameraController == null || !cameraController.value.isInitialized) {
      return;
    }
    if (state == AppLifecycleState.inactive) {
      cameraController.dispose();
    } else if (state == AppLifecycleState.resumed) {
      _initializeCamera();
    }
  }


  @override
  Widget build(BuildContext context) {
    if (_cameraController == null || !_cameraController!.value.isInitialized) {
      return Center(child: CircularProgressIndicator());
    }

    return Scaffold(
      appBar: AppBar(title: Text('Face Liveness Detection')),
      body: Column(
        children: [

          // Captured Image Preview
          if (_capturedImage != null)
            Expanded(
              child: Image.file(File(_capturedImage!.path)),
            ),
          // Current Direction
          if (_capturedImage == null)
          Padding(
            padding: const EdgeInsets.all(8.0),
            child: Text(
              _getCurrentDirection(),
              style: TextStyle(fontSize: 13, fontWeight: FontWeight.bold),
            ),
          ),
          // Camera Preview (300x300)
          if (_capturedImage == null)
            Container(

            width: 300,
            height: 300,
            child: ClipRRect(
              borderRadius: BorderRadius.circular(10),
              child: CameraPreview(_cameraController!),
            ),
          ),
          // List of Successful Directions
          Expanded(
            child: ListView.builder(
              itemCount: _successfulSteps.length,
              itemBuilder: (context, index) {
                return ListTile(
                  leading: Icon(Icons.check_circle, color: Colors.green),
                  title: Text(_successfulSteps[index], style: TextStyle(color: Colors.black),),
                );
              },
            ),
          ),
        ],
      ),
    );
  }
}


want to making an video content based on my code. Please give me this content
