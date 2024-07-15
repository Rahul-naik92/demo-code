class Debouncer {
  final int milliseconds;
  VoidCallback? action;
  Timer? _timer;

  Debouncer({required this.milliseconds});

  run(VoidCallback action) {
    if (_timer != null) {
      _timer!.cancel();
    }
    _timer = Timer(Duration(milliseconds: milliseconds), action);
  }

  void debounce(String tag, Duration duration, Function() action) {
    run(action);
  }
}

class Preloader extends StatefulWidget {
  @override
  _PreloaderState createState() => _PreloaderState();
}

class _PreloaderState extends State<Preloader> {
  final List<String> listOfUrls = [
    "https://livepin-admin-bucket.s3.ap-south-1.amazonaws.com/4_461f6b13fd.mp4",
    "https://livepin-admin-bucket.s3.ap-south-1.amazonaws.com/1_9afae56794.mp4",
    "https://livepin-admin-bucket.s3.ap-south-1.amazonaws.com/4_461f6b13fd.mp4",
    "https://livepin-admin-bucket.s3.ap-south-1.amazonaws.com/4_461f6b13fd.mp4",
    "https://livepin-admin-bucket.s3.ap-south-1.amazonaws.com/1_9afae56794.mp4",
    "https://livepin-admin-bucket.s3.ap-south-1.amazonaws.com/4_461f6b13fd.mp4",
    "https://livepin-admin-bucket.s3.ap-south-1.amazonaws.com/4_461f6b13fd.mp4",
    "https://livepin-admin-bucket.s3.ap-south-1.amazonaws.com/1_9afae56794.mp4",
    "https://livepin-admin-bucket.s3.ap-south-1.amazonaws.com/4_461f6b13fd.mp4",
  ];

  final List<VideoPlayerController> videoPlayerController = [];
  final Logger logger = Logger();
  int currentIndex = 0;
  final Debouncer debouncer = Debouncer(milliseconds: 200);
  int focusedIndex = 0;
  bool _isPlaying = false;

  @override
  void initState() {
    super.initState();
    for (var i = 0; i < listOfUrls.length; i++) {
      videoPlayerController.add(VideoPlayerController.network(listOfUrls[i]));
    }
    _initializeControllerAtIndex(0);
    _initializeControllerAtIndex(1);
  }

  void _playNextReel(int index) {
    _stopControllerAtIndex(index - 1);
    _disposeControllerAtIndex(index - 2);
    _playControllerAtIndex(index);
    _initializeControllerAtIndex(index + 1);
  }

  void _playPreviousReel(int index) {
    _stopControllerAtIndex(index + 1);
    _disposeControllerAtIndex(index + 2);
    _playControllerAtIndex(index);
    _initializeControllerAtIndex(index - 1);
  }

  void _stopControllerAtIndex(int index) {
    if (index >= 0 && index < listOfUrls.length) {
      final _controller = videoPlayerController[index];
      _controller.pause();
      _controller.seekTo(const Duration());
      logger.w('stopped $index');
    }
  }

  void _disposeControllerAtIndex(int index) {
    if (index >= 0 && index < listOfUrls.length) {
      final _controller = videoPlayerController[index];
      _controller.dispose();
      logger.w('Disposed $index');
    }
  }

  void _playControllerAtIndex(int index) {
    if (index >= 0 && index < listOfUrls.length) {
      final _controller = videoPlayerController[index];
      if (_controller.value.isInitialized) {
        _controller.play();
        _controller.setLooping(true);
      } else {
        _initializeControllerAtIndex(index).then((_) {
          videoPlayerController[index].play();
          videoPlayerController[index].setLooping(true);
          setState(() {});
        });
      }
      logger.w('playing $index');
    }
  }

  Future<void> _initializeControllerAtIndex(int index) async {
    if (index >= 0 && index < listOfUrls.length) {
      final _controller = VideoPlayerController.network(listOfUrls[index]);
      videoPlayerController[index] = _controller;
      await _controller.initialize().then((_) {
        setState(() {});
      });
      logger.w('initialized $index');
    }
  }

  void _onIndexChanged(int index) {
    setState(() {
      _isPlaying = true;
      currentIndex = index;
    });
    if (index > focusedIndex) {
      debouncer.run(() async {
        _playNextReel(index);
      });
    } else {
      debouncer.run(() async {
        _playPreviousReel(index);
      });
    }
    focusedIndex = index;
  }

  void _pause() {
    setState(() {
      _isPlaying = false;
    });
  }

  void _play() {
    setState(() {
      _isPlaying = true;
    });
  }

  @override
  void dispose() {
    for (var controller in videoPlayerController) {
      controller.dispose();
    }
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Swiper(
        physics: videoPlayerController.isNotEmpty &&
                videoPlayerController[currentIndex].value.isInitialized
            ? const AlwaysScrollableScrollPhysics()
            : const NeverScrollableScrollPhysics(),
        loop: false,
        scrollDirection: Axis.vertical,
        itemCount: videoPlayerController.length,
        onIndexChanged: _onIndexChanged,
        itemBuilder: (context, index) {
          return focusedIndex == index
              ? videoPlayerController[index].value.isInitialized
                  ? FadeIn(
                      duration: const Duration(milliseconds: 900),
                      child: GestureDetector(
                        onTap: () {
                          if (_isPlaying) {
                            videoPlayerController[index].pause();
                            _pause();
                          } else {
                            videoPlayerController[index].play();
                            videoPlayerController[index].setLooping(true);
                            _play();
                          }
                        },
                        child: ClipRRect(
                          borderRadius: BorderRadius.circular(6),
                          child: Container(
                            child: videoPlayerController[index]
                                    .value
                                    .isInitialized
                                ? AspectRatio(
                                    aspectRatio: videoPlayerController[index]
                                        .value
                                        .aspectRatio,
                                    child: VideoPlayer(
                                        videoPlayerController[index]),
                                  )
                                : Container(),
                          ),
                        ),
                      ),
                    )
                  : const Center(child: CircularProgressIndicator())
              : const Center(child: CircularProgressIndicator());
        },
      ),
    );
  }
}
