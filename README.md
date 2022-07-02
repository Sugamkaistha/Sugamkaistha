
import 'dart:html';
import 'dart:math';
import 'dart:ui' as ui;

import 'package:flutter/material.dart';
import 'package:flutter/painting.dart';

void main() => runApp(SpotifyApp());

class SpotifyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      debugShowCheckedModeBanner: false,
      home: MainScreen(),
    );
  }
}

class MainScreen extends StatefulWidget {
  @override
  _MainScreenState createState() => _MainScreenState();
}

class _MainScreenState extends State<MainScreen> {
  static int audioId = 0;
  final _audio = Utils.audio(null, audioId, true);
  final _audioKey = GlobalKey<PlayerSectionState>();

  @override
  Widget build(BuildContext context) {
    final isMobile = MediaQuery.of(context).size.width < 650;
    return Material(
      child: isMobile
          ? MobileSection(audio: _audio, audioKey: _audioKey)
          : HomeSection(audio: _audio, audioKey: _audioKey),
    );
  }
}

class MobileSection extends StatefulWidget {
  const MobileSection({Key key, this.audio, this.audioKey}) : super(key: key);

  final Tuple<Widget, AudioElement> audio;
  final GlobalKey<PlayerSectionState> audioKey;

  @override
  _MobileSectionState createState() => _MobileSectionState();
}

class _MobileSectionState extends State<MobileSection> with SingleTickerProviderStateMixin {
  Animation<double> _fadeAnimation;
  AnimationController _fadeController;

  GenericClick<SectionType> _genericClick;

  Widget body;
  List<Widget> _bodies = [];

  final listKey = GlobalKey<ListScreenState>();

  @override
  void initState() {
    super.initState();
    _homeClick();
    _setupAnimation();
    Future.microtask(() {
      _bodies.add(HomeScreen(isMobile: true, click: _genericClick));
      _bodies.add(ListScreen(isMobile: true, key: listKey, click: _genericClick));
      setState(() => body = _bodies[0]);
    });
  }

  void _setupAnimation() {
    _fadeController = AnimationController(duration: Duration(milliseconds: 300), vsync: this);
    _fadeAnimation = Tween<double>(begin: 0, end: 1).animate(_fadeController);
    _fadeController.forward(from: 1);
  }

  void _homeClick() {
    _genericClick = (section, item) {
      switch (section) {
        case SectionType.BROWSE:
          _fadeController.forward(from: 0);
          setState(() => body = _bodies[0]);
          break;
        case SectionType.ALBUM:
          _fadeController.forward(from: 0);
          setState(() => body = _bodies[1]);
          Future.delayed(Duration(milliseconds: 100), () {
            listKey.currentState.setPlaylist(item);
          });
          break;
        case SectionType.HOME:
          // TODO: Handle this case.
          break;
        case SectionType.BROWSE_PODCAST:
          // TODO: Handle this case.
          break;
        case SectionType.PODCAST:
          // TODO: Handle this case.
          break;
        case SectionType.GRID:
          // TODO: Handle this case.
          break;
        case SectionType.GRID_CIRCLE:
          // TODO: Handle this case.
          break;
        case SectionType.GRID_SECTION:
          // TODO: Handle this case.
          break;
        case SectionType.LIST:
          // TODO: Handle this case.
          break;
        case SectionType.LIST_IMAGE:
          // TODO: Handle this case.
          break;
        case SectionType.ARTIST:
          // TODO: Handle this case.
          break;
        case SectionType.SONG:
          // TODO: Handle this case.
          break;
        case SectionType.AUDIO:
          Utils.currentSong = item;
          handlePlayer(item);
          break;
        case SectionType.FULLSCREEN:
          break;
      }
    };
  }

  void handlePlayer(Song song) {
    if (widget.audio.second.currentSrc == song.url) {
      if (widget.audio.second.paused) {
        widget.audio.second.play();
        window.navigator.mediaSession.setActionHandler('play', () {
          window.navigator.mediaSession.playbackState = "playing";
        });
      } else {
        widget.audio.second.pause();
        window.navigator.mediaSession.setActionHandler('pause', () {
          window.navigator.mediaSession.playbackState = "paused";
        });
      }
      widget.audioKey.currentState.refresh(song);
    } else {
      widget.audio.second.src = song.url;
      widget.audio.second.currentTime = 0.0;
      widget.audio.second.play();
      window.navigator.mediaSession.metadata = MediaMetadata()
        ..title = song.name
        ..artist = song.artist.name
        ..album = song.album.name
        ..artwork = [Src(song.image)];
      window.navigator.mediaSession.setActionHandler('play', () {
        widget.audio.second.play();
        window.navigator.mediaSession.playbackState = "playing";
      });
      window.navigator.mediaSession.setActionHandler('pause', () {
        widget.audio.second.pause();
        window.navigator.mediaSession.playbackState = "paused";
      });
      widget.audio.second.onLoadedData.listen((event) {
        widget.audioKey.currentState.refresh(song);
      });
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: FadeTransition(opacity: _fadeAnimation, child: body),
      bottomNavigationBar: Column(
        mainAxisSize: MainAxisSize.min,
        children: [
          PlayerSection(key: widget.audioKey, audio: widget.audio.second, isMobile: true),
          Line(horizontal: true, color: Colors.black),
          BottomNavigationBar(
            items: [
              BottomNavigationBarItem(icon: Icon(Icons.home), title: Text('Home')),
              BottomNavigationBarItem(icon: Icon(Icons.search), title: Text('Search')),
              BottomNavigationBarItem(icon: Icon(Icons.library_books), title: Text('Your Library')),
            ],
            elevation: 0,
            backgroundColor: Utils.playerSectionBg,
            type: BottomNavigationBarType.fixed,
            selectedItemColor: Colors.white,
            unselectedItemColor: Colors.grey,
            onTap: (index) {
              switch (index) {
                case 0:
                  _genericClick(SectionType.BROWSE, null);
                  break;
              }
            },
          ),
        ],
      ),
    );
  }
}

class HomeSection extends StatefulWidget {
  const HomeSection({Key key, this.audio, this.audioKey}) : super(key: key);

  final Tuple<Widget, AudioElement> audio;
  final GlobalKey<PlayerSectionState> audioKey;

  @override
  _HomeSectionState createState() => _HomeSectionState();
}

class _HomeSectionState extends State<HomeSection> {
  GenericClick<SectionType> _genericClick;
  ItemClick _itemClick;
  bool _switchDesktop = true;
  int index = 0;

  bool _fullScreen = false;
  Widget body = Container();
  List<Widget> _bodies = [];

  final listKey = GlobalKey<ListScreenState>();

  @override
  void initState() {
    super.initState();
    Future.microtask(homeClick);
    Future.microtask(itemClick);
    Future.microtask(() {
      _bodies.add(HomeScreen(isMobile: false, click: _genericClick));
      _bodies.add(ListScreen(isMobile: false, key: listKey, click: _genericClick));
      setState(() => body = _bodies[0]);
    });
  }

  void homeClick() {
    _genericClick = (section, item) {
      switch (section) {
        case SectionType.BROWSE:
          setState(() => body = _bodies[0]);
          break;
        case SectionType.ALBUM:
          setState(() => body = _bodies[1]);
          Future.delayed(Duration(milliseconds: 100), () {
            listKey.currentState.setPlaylist(item);
          });
          break;
        case SectionType.HOME:
          // TODO: Handle this case.
          break;
        case SectionType.BROWSE_PODCAST:
          // TODO: Handle this case.
          break;
        case SectionType.PODCAST:
          // TODO: Handle this case.
          break;
        case SectionType.GRID:
          // TODO: Handle this case.
          break;
        case SectionType.GRID_CIRCLE:
          // TODO: Handle this case.
          break;
        case SectionType.GRID_SECTION:
          // TODO: Handle this case.
          break;
        case SectionType.LIST:
          // TODO: Handle this case.
          break;
        case SectionType.LIST_IMAGE:
          // TODO: Handle this case.
          break;
        case SectionType.ARTIST:
          // TODO: Handle this case.
          break;
        case SectionType.SONG:
          // TODO: Handle this case.
          break;
        case SectionType.AUDIO:
          Utils.currentSong = item;
          handlePlayer(item);
          break;
        case SectionType.FULLSCREEN:
          setState(() => _fullScreen = item);
          if (_fullScreen) {
            document.documentElement.requestFullscreen();
          } else {
            setState(() => body = _bodies[0]);
            document.exitFullscreen();
          }
          break;
      }
    };
  }

  void handlePlayer(Song song) {
    if (widget.audio.second.currentSrc == song.url) {
      if (widget.audio.second.paused) {
        widget.audio.second.play();
      } else {
        widget.audio.second.pause();
      }
      widget.audioKey.currentState.refresh(song);
    } else {
      widget.audio.second.pause();
      widget.audio.second.src = song.url;
      widget.audio.second.load();
      window.navigator.mediaSession.metadata = MediaMetadata()
        ..title = song.name
        ..artist = song.artist.name
        ..album = song.album.name
        ..artwork = [Src(song.image)];
      window.navigator.mediaSession.setActionHandler('play', () {
        widget.audio.second.play();
        window.navigator.mediaSession.playbackState = "playing";
      });
      window.navigator.mediaSession.setActionHandler('pause', () {
        widget.audio.second.pause();
        window.navigator.mediaSession.playbackState = "paused";
      });
      widget.audio.second.onLoadedData.listen((event) {
        widget.audioKey.currentState.refresh(song);
        widget.audioKey.currentState.updateTime();
      });
    }
  }

  void itemClick() {
    setState(() => _itemClick = (_index) => index = _index);
    selectItem();
  }

  void setIndex() {
    Future.microtask(() {
      _switchDesktop = true;
      selectItem();
    });
  }

  void selectItem() {
    Utils.itemSelection(Utils.leftSectionItems, false);
    if (Utils.leftSectionItems[index].key.currentState != null) {
      Utils.leftSectionItems[index].key.currentState.selected(true);
    }
  }

  @override
  Widget build(BuildContext context) {
    final size = MediaQuery.of(context).size;
    final isDesktop = MediaQuery.of(context).size.width > 1000;
    final isMobile = MediaQuery.of(context).size.width < 650;
    if (isMobile) _switchDesktop = false;
    if (!_switchDesktop && !isMobile) setIndex();
    return Column(
      children: [
        Expanded(
          child: !_fullScreen
              ? Row(
                  children: [
                    if (!isMobile) LeftSection(itemClick: _itemClick),
                    Expanded(child: body),
                    if (isDesktop) RightSection(),
                  ],
                )
              : Container(
                  decoration: BoxDecoration(
                    gradient: LinearGradient(
                      colors: [
                        Colors.black,
                        Color(0xFF626262),
                        Colors.black,
                      ],
                      begin: Alignment.topCenter,
                      end: Alignment.bottomCenter,
                    ),
                  ),
                  child: Column(
                    crossAxisAlignment: CrossAxisAlignment.center,
                    children: [
                      Container(
                        width: size.width,
                        alignment: Alignment.centerRight,
                        margin: const EdgeInsets.only(right: 60, top: 40),
                        child: GestureDetector(
                          onTap: () {
                            _genericClick(SectionType.FULLSCREEN, false);
                            document.exitFullscreen();
                          },
                          child: Container(
                            height: 50,
                            width: 50,
                            decoration: BoxDecoration(
                              color: Colors.black,
                              borderRadius: BorderRadius.circular(25),
                              border: Border.all(color: Colors.white, width: 1),
                            ),
                            child: Center(
                              child: Icon(Icons.fullscreen_exit, color: Colors.white, size: 40),
                            ),
                          ),
                        ),
                      ),
                      Space(size: 60),
                      Expanded(
                        child: AspectRatio(
                          aspectRatio: 1,
                          child: Card(
                            elevation: 10,
                            shape: RoundedRectangleBorder(),
                            child: Image.network(Utils.currentSong.image, fit: BoxFit.cover),
                          ),
                        ),
                      ),
                      Space(size: 20),
                      Text(
                        Utils.currentSong.name,
                        style: TextStyle(color: Colors.white, fontWeight: FontWeight.w600, fontSize: 40),
                      ),
                      Space(size: 8),
                      Text(
                        Utils.currentSong.artist.name,
                        style: TextStyle(color: Colors.grey, fontWeight: FontWeight.w600, fontSize: 25),
                      ),
                      Space(size: 60),
                    ],
                  ),
                ),
        ),
        PlayerSection(click: _genericClick, key: widget.audioKey, audio: widget.audio.second, isMobile: false)
      ],
    );
  }
}

class LeftSection extends StatefulWidget {
  const LeftSection({Key key, this.itemClick}) : super(key: key);

  final ItemClick itemClick;

  @override
  LeftSectionState createState() => LeftSectionState();
}

class LeftSectionState extends State<LeftSection> {
  @override
  void initState() {
    super.initState();
    Future.microtask(() => Utils.leftSectionItems[0].selected = true);
  }

  @override
  Widget build(BuildContext context) {
    return Container(
      width: 180,
      color: Utils.leftPanelBg,
      padding: const EdgeInsets.only(top: 12),
      child: Column(
        children: [
          WindowButtons(),
          Space(size: 30),
          LeftListItem(
            key: Utils.leftSectionItems[0].key,
            icon: Icons.home,
            item: Utils.leftSectionItems[0],
            itemClick: widget.itemClick,
            index: 0,
          ),
          Space(size: 12),
          LeftListItem(
            key: Utils.leftSectionItems[1].key,
            icon: Icons.open_in_browser,
            item: Utils.leftSectionItems[1],
            itemClick: widget.itemClick,
            index: 1,
          ),
          Space(size: 12),
          LeftListItem(
            key: Utils.leftSectionItems[2].key,
            icon: Icons.radio,
            item: Utils.leftSectionItems[2],
            itemClick: widget.itemClick,
            index: 2,
          ),
          Space(size: 40),
          Expanded(
            child: ListView.builder(
              itemBuilder: (context, i) {
                final _index = i + 3;
                final item = Utils.leftSectionItems[_index];
                return LeftListItem(key: item.key, item: item, itemClick: widget.itemClick, index: _index);
              },
              itemCount: Utils.leftSectionItems.length - 3,
            ),
          ),
          Line(horizontal: true),
          AddPlayList(),
        ],
      ),
    );
  }
}

class RightSection extends StatefulWidget {
  @override
  _RightSectionState createState() => _RightSectionState();
}

class _RightSectionState extends State<RightSection> {
  @override
  Widget build(BuildContext context) {
    return Container(
      width: 250,
      color: Utils.rightPanelBg,
      child: ListView(
        padding: const EdgeInsets.symmetric(horizontal: 15, vertical: 10),
        children: [
          Text(
            'Friend Activity',
            style: TextStyle(color: Colors.white, fontWeight: FontWeight.w600, fontSize: 20),
          ),
          Space(size: 10),
          Container(height: 1, color: Colors.grey[800]),
          Space(size: 15),
          ...Utils.users.map((item) => FriendsItem(user: item)),
          Padding(
            padding: const EdgeInsets.symmetric(horizontal: 20),
            child: OutlineButton(
              onPressed: () {},
              highlightedBorderColor: Colors.grey[700],
              borderSide: BorderSide(color: Colors.white, width: 2.5),
              shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(30)),
              child: Text('FIND FRIENDS', style: TextStyle(color: Colors.white, fontSize: 12)),
            ),
          ),
        ],
      ),
    );
  }
}

class FriendsItem extends StatelessWidget {
  const FriendsItem({Key key, this.user}) : super(key: key);

  final User user;

  @override
  Widget build(BuildContext context) {
    return Container(
      padding: const EdgeInsets.only(bottom: 30),
      child: Row(
        crossAxisAlignment: CrossAxisAlignment.center,
        children: [
          Container(
            height: 40,
            width: 40,
            child: CircleAvatar(
              backgroundColor: Colors.grey[800],
              child: ClipRRect(
                borderRadius: BorderRadius.circular(20),
                child: Image.network(user.image, fit: BoxFit.cover),
              ),
            ),
          ),
          Space(size: 15, horizontal: true),
          Expanded(
            child: Column(
              crossAxisAlignment: CrossAxisAlignment.start,
              children: [
                Row(
                  children: [
                    Expanded(
                      child: Text(
                        user.name,
                        maxLines: 1,
                        overflow: TextOverflow.ellipsis,
                        style: TextStyle(color: Colors.white, fontWeight: FontWeight.w600, fontSize: 15),
                      ),
                    ),
                    Space(size: 5, horizontal: true),
                    Text(user.time, style: TextStyle(color: Colors.grey, fontSize: 12)),
                  ],
                ),
                Space(size: 5),
                Padding(
                  padding: const EdgeInsets.only(right: 20),
                  child: Text(user.song, style: TextStyle(color: Colors.grey, fontSize: 12)),
                ),
                Space(size: 5),
                Padding(
                  padding: const EdgeInsets.only(right: 20),
                  child: Text(user.playlist, style: TextStyle(color: Colors.grey, fontSize: 11)),
                ),
              ],
            ),
          ),
        ],
      ),
    );
  }
}

class PlayerSection extends StatefulWidget {
  const PlayerSection({Key key, this.click, this.audio, this.isMobile}) : super(key: key);

  final GenericClick<SectionType> click;
  final AudioElement audio;
  final bool isMobile;

  @override
  PlayerSectionState createState() => PlayerSectionState();
}

class PlayerSectionState extends State<PlayerSection> {
  Song _song;
  double _current = 0.0;
  var _volume = 1.0;
  bool _disposed = false;
  bool _paused = false;

  void refresh(Song song) => setState(() => _song = song);

  void updateTime() {
    widget.audio.onTimeUpdate.listen((event) {
      if (!_disposed) {
        setState(() => _current = widget.audio.currentTime);
      }
    });
  }

  @override
  void initState() {
    super.initState();
    if (mounted) {
      Future.microtask(() {
        widget.audio.onPlay.listen((event) {
          if (!_disposed) {
            setState(() => _paused = false);
          }
        });
        widget.audio.onPause.listen((event) {
          if (!_disposed) {
            setState(() => _paused = true);
          }
        });
      });
    }
  }

  @override
  void dispose() {
    _disposed = true;
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    final size = MediaQuery.of(context).size;
    return GestureDetector(
      onTap: widget.isMobile
          ? () {
              Navigator.of(context).push(
                MaterialPageRoute(
                  builder: (_) => PlayerScreen(audio: widget.audio, song: _song),
                  fullscreenDialog: true,
                ),
              );
            }
          : null,
      child: Container(
        color: Utils.playerSectionBg,
        child: widget.isMobile
            ? AnimatedContainer(
                duration: Duration(milliseconds: 200),
                height: _song != null ? 70 : 0,
                child: Row(
                  children: [
                    Container(
                        width: 70,
                        height: 70,
                        child: Image.network(_song != null ? _song.image : '', fit: BoxFit.cover)),
                    Space(size: 20, horizontal: true),
                    Expanded(
                      child: Column(
                        crossAxisAlignment: CrossAxisAlignment.start,
                        mainAxisAlignment: MainAxisAlignment.center,
                        children: [
                          Text(_song != null ? '${_song.name} - ${_song.artist.name}' : '',
                              style: TextStyle(color: Colors.white), maxLines: 1, overflow: TextOverflow.ellipsis),
                          Space(size: 8),
                          Row(
                            children: [
                              Icon(Icons.devices, color: Colors.white, size: 14),
                              Space(size: 5, horizontal: true),
                              Expanded(
                                child: Text('Available Devices', style: TextStyle(color: Colors.white, fontSize: 12),
                                  maxLines: 1, overflow: TextOverflow.ellipsis),
                              ),
                            ],
                          )
                        ],
                      ),
                    ),
                    Space(size: 20, horizontal: true),
                    Icon(Icons.favorite_border, color: Colors.white),
                    Space(size: 20, horizontal: true),
                    GestureDetector(
                      onTap: handlePlayer,
                      child: Icon(_paused ? Icons.play_arrow : Icons.pause, color: Colors.white, size: 30),
                    ),
                    Space(size: 20, horizontal: true),
                  ],
                ),
              )
            : Container(
                height: 90,
                width: size.width,
                color: Utils.playerSectionBg,
                child: Padding(
                  padding: const EdgeInsets.symmetric(horizontal: 15),
                  child: Row(
                    mainAxisAlignment: MainAxisAlignment.center,
                    mainAxisSize: MainAxisSize.max,
                    children: [
                      Flexible(
                        flex: 1,
                        fit: FlexFit.tight,
                        child: _song != null
                            ? Row(
                                mainAxisAlignment: MainAxisAlignment.start,
                                children: [
                                  Padding(
                                    padding: const EdgeInsets.only(right: 15),
                                    child: SizedBox(
                                      height: 65,
                                      width: 65,
                                      child: Image.network(_song.image, fit: BoxFit.cover),
                                    ),
                                  ),
                                  Column(
                                    crossAxisAlignment: CrossAxisAlignment.start,
                                    mainAxisAlignment: MainAxisAlignment.center,
                                    children: [
                                      Row(
                                        crossAxisAlignment: CrossAxisAlignment.start,
                                        mainAxisAlignment: MainAxisAlignment.start,
                                        children: [
                                          Text(_song.name,
                                              style: TextStyle(color: Colors.white, fontWeight: FontWeight.w600)),
                                          Space(size: 10, horizontal: true),
                                          Icon(Icons.favorite_border, color: Colors.grey, size: 15),
                                        ],
                                      ),
                                      Text(_song.artist.name, style: TextStyle(color: Colors.grey)),
                                    ],
                                  )
                                ],
                              )
                            : SizedBox(),
                      ),
                      Flexible(
                        flex: 2,
                        child: Column(
                          mainAxisAlignment: MainAxisAlignment.center,
                          children: [
                            Flexible(
                              flex: 2,
                              child: Row(
                                crossAxisAlignment: CrossAxisAlignment.center,
                                mainAxisAlignment: MainAxisAlignment.center,
                                children: [
                                  Icon(Icons.shuffle, color: Colors.white, size: 15),
                                  Space(size: 20, horizontal: true),
                                  Icon(Icons.skip_previous, color: Colors.white, size: 18),
                                  Space(size: 20, horizontal: true),
                                  GestureDetector(
                                    onTap: _song != null
                                        ? () => widget.audio.paused ? widget.audio.play() : widget.audio.pause()
                                        : null,
                                    child: Icon(
                                        widget.audio.paused ? Icons.play_circle_filled : Icons.pause_circle_filled,
                                        color: Colors.white,
                                        size: 40),
                                  ),
                                  Space(size: 20, horizontal: true),
                                  Icon(Icons.skip_next, color: Colors.white, size: 18),
                                  Space(size: 20, horizontal: true),
                                  Icon(Icons.repeat, color: Colors.white, size: 15),
                                ],
                              ),
                            ),
                            Flexible(
                              flex: 1,
                              child: Row(
                                mainAxisSize: MainAxisSize.min,
                                mainAxisAlignment: MainAxisAlignment.center,
                                children: [
                                  Space(size: 20, horizontal: true),
                                  if (_song != null)
                                    Container(
                                      width: 35,
                                      child: Text(Utils.parseTime(_current),
                                          style: TextStyle(color: Colors.white, fontSize: 12)),
                                    ),
                                  Expanded(
                                    child: SliderTheme(
                                      data: SliderThemeData(
                                        trackHeight: 6,
                                        overlayShape: RoundSliderOverlayShape(overlayRadius: 0),
                                        thumbShape:
                                            RoundSliderThumbShape(enabledThumbRadius: 6, disabledThumbRadius: 6),
                                        thumbColor: Colors.white,
                                        activeTrackColor: Colors.green,
                                      ),
                                      child: Slider(
                                        value: _song != null ? _current : 0,
                                        activeColor: Colors.white,
                                        inactiveColor: Colors.grey[800],
                                        max: _song != null && !widget.audio.duration.isNaN
                                            ? widget.audio.duration
                                            : 1,
                                        onChanged: _song != null
                                            ? (value) {
                                                setState(() {
                                                  _current = value;
                                                  widget.audio.currentTime = _current;
                                                });
                                              }
                                            : null,
                                      ),
                                    ),
                                  ),
                                  if (_song != null)
                                    Text(Utils.parseTime(!widget.audio.duration.isNaN ? widget.audio.duration : 0),
                                        style: TextStyle(color: Colors.white, fontSize: 12)),
                                  Space(size: 20, horizontal: true),
                                ],
                              ),
                            ),
                          ],
                        ),
                      ),
                      Flexible(
                        flex: 1,
                        fit: FlexFit.tight,
                        child: _song != null
                            ? Row(
                                mainAxisAlignment: MainAxisAlignment.end,
                                children: [
                                  Icon(Icons.playlist_play, color: Colors.white, size: 18),
                                  Space(size: 10, horizontal: true),
                                  Icon(Icons.devices, color: Colors.white, size: 15),
                                  Space(size: 12, horizontal: true),
                                  Icon(Icons.volume_up, color: Colors.white, size: 15),
                                  Space(size: 10, horizontal: true),
                                  Container(
                                    width: 50,
                                    child: SliderTheme(
                                      data: SliderThemeData(
                                        trackHeight: 4,
                                        overlayShape: RoundSliderOverlayShape(overlayRadius: 0),
                                        thumbShape:
                                            RoundSliderThumbShape(enabledThumbRadius: 5, disabledThumbRadius: 0),
                                        thumbColor: Colors.white,
                                        activeTrackColor: Colors.green,
                                      ),
                                      child: Slider(
                                        value: _volume,
                                        max: 1,
                                        onChanged: (value) {
                                          setState(() {
                                            widget.audio.volume = value;
                                            _volume = value;
                                          });
                                        },
                                      ),
                                    ),
                                  ),
                                  Space(size: 10, horizontal: true),
                                  GestureDetector(
                                    onTap: () {
                                      widget.click(SectionType.FULLSCREEN, window.innerHeight != window.screen.height);
                                    },
                                    child: Icon(
                                        window.innerHeight == window.screen.height
                                            ? Icons.fullscreen_exit
                                            : Icons.fullscreen,
                                        color: Colors.white,
                                        size: 15),
                                  ),
                                ],
                              )
                            : SizedBox(),
                      )
                    ],
                  ),
                ),
              ),
      ),
    );
  }

  void handlePlayer() {
    setState(() {
      if (widget.audio.paused) {
        widget.audio.play();
        _paused = false;
      } else {
        widget.audio.pause();
        _paused = true;
      }
    });
  }
}

class WindowButtons extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Row(
      children: [
        Space(size: 20, horizontal: true),
        ClipOval(child: Container(color: Colors.red, width: 12, height: 12)),
        Space(size: 10, horizontal: true),
        ClipOval(child: Container(color: Colors.yellow, width: 12, height: 12)),
        Space(size: 10, horizontal: true),
        ClipOval(child: Container(color: Colors.green, width: 12, height: 12)),
      ],
    );
  }
}

class LeftListItem extends StatefulWidget {
  const LeftListItem({Key key, this.itemClick, this.icon, this.item, this.index}) : super(key: key);

  final ItemClick itemClick;
  final IconData icon;
  final Item item;
  final int index;

  @override
  LeftListItemState createState() => LeftListItemState();
}

class LeftListItemState extends State<LeftListItem> {
  bool _selected = false;
  bool _hover = false;

  void selected(bool selected) => setState(() => _selected = selected);

  @override
  Widget build(BuildContext context) {
    return widget.item.text != null
        ? GestureDetector(
            onTap: () {
              Utils.itemSelection(Utils.leftSectionItems, false);
              selected(true);
              widget.itemClick(widget.index);
            },
            child: MouseRegion(
              onHover: (_) => setState(() => _hover = true),
              onExit: (_) => setState(() => _hover = false),
              child: Container(
                padding: EdgeInsets.only(
                    top: when(widget.item.type, {
                      TextType.HEADER: 0,
                      TextType.NORMAL: 8,
                      TextType.TITLE: 4,
                      TextType.BOLD: 8,
                    }),
                    bottom: when(widget.item.type, {
                      TextType.HEADER: 0,
                      TextType.NORMAL: 8,
                      TextType.TITLE: 4,
                      TextType.BOLD: 8,
                    }),
                    right: 10),
                child: Row(
                  children: [
                    if (_selected)
                      Container(
                          width: 4,
                          color: Utils.selectedItem,
                          height: when(widget.item.type, {
                            TextType.HEADER: 20,
                            TextType.NORMAL: 15,
                            TextType.TITLE: 0,
                            TextType.BOLD: 15,
                          })),
                    if (widget.item.type == TextType.HEADER) Space(size: _selected ? 16 : 20, horizontal: true),
                    if (widget.icon != null)
                      Icon(widget.icon, color: _selected ? Colors.white : _hover ? Colors.white : Colors.grey),
                    if (widget.icon == null) Space(size: _selected ? 16 : 20, horizontal: true),
                    if (widget.icon != null) Space(size: 15, horizontal: true),
                    Text(widget.item.text,
                        maxLines: 1,
                        overflow: TextOverflow.ellipsis,
                        style: TextStyle(
                            color: widget.item.type == TextType.TITLE
                                ? Colors.grey
                                : _selected ? Colors.white : _hover ? Colors.white : Colors.grey,
                            fontWeight: when(widget.item.type, {
                              TextType.HEADER: FontWeight.w600,
                              TextType.NORMAL: FontWeight.w400,
                              TextType.TITLE: FontWeight.w300,
                              TextType.BOLD: FontWeight.w600,
                            }),
                            fontSize: when(widget.item.type, {
                              TextType.HEADER: 14,
                              TextType.NORMAL: 14,
                              TextType.TITLE: 12,
                              TextType.BOLD: 14,
                            })))
                  ],
                ),
              ),
            ),
          )
        : Space(size: 20);
  }
}

class HomeScreen extends StatefulWidget {
  const HomeScreen({Key key, this.isMobile, this.click}) : super(key: key);

  final GenericClick<SectionType> click;
  final bool isMobile;

  @override
  _HomeScreenState createState() => _HomeScreenState();
}

class _HomeScreenState extends State<HomeScreen> {
  final _controller = ScrollController();
  var _opacity = 1.0;
  var _barOpacity = 0.0;
  var _height = 0.0;

  @override
  void initState() {
    super.initState();
    _controller.addListener(() {
      setState(() {
        if (_controller.offset < 51) {
          _barOpacity = (_controller.offset - 0.0) / (50.0 - 0.0);
          _height = (_controller.offset - 0.0) / (50.0 - 0.0);
        } else if (_controller.offset > 50) {
          _barOpacity = 1.0;
          _height = 1.0;
        }
      });
      if (_opacity == 1.0 && _controller.offset > 20) {
        setState(() => _opacity = 0.0);
      } else if (_opacity == 0.0 && _controller.offset < 20) {
        setState(() => _opacity = 1.0);
      }
    });
  }

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    final size = MediaQuery.of(context).size;
    return Container(
      width: size.width,
      height: size.height,
      decoration: BoxDecoration(
        gradient: LinearGradient(
          colors: [
            Color(0xFF404040),
            Color(0xFF181818),
            Color(0xFF181818),
            Color(0xFF181818),
            Color(0xFF181818),
          ],
          begin: Alignment(-0.4, -1.2),
          end: Alignment.bottomCenter,
        ),
      ),
      child: Stack(
        children: [
          if (widget.isMobile)
            AnimatedOpacity(
              duration: Duration(milliseconds: 200),
              opacity: _opacity,
              child: MobileAppBar(showMenu: true, icon: Icons.settings),
            ),
          ListView(
            controller: _controller,
            padding: EdgeInsets.only(top: widget.isMobile ? 10 : 140, bottom: 10),
            children: [
              ...Utils.homeList
                  .asMap()
                  .entries
                  .map((item) => PlaylistItem(click: widget.click, playlist: item.value, small: item.key == 0))
            ],
          ),
          if (!widget.isMobile)
            Container(
              height: 170 - (_height * 40),
              color: Color(0xFF121212).withOpacity(_barOpacity > 0.6 ? 1 : _barOpacity < 0.1 ? 0 : _barOpacity),
              child: Padding(
                padding: const EdgeInsets.symmetric(horizontal: 5),
                child: Column(
                  crossAxisAlignment: CrossAxisAlignment.start,
                  children: [
                    Space(size: 5),
                    Row(
                      children: [
                        Icon(Icons.keyboard_arrow_left, color: Colors.grey, size: 35),
                        Space(size: 5, horizontal: true),
                        Icon(Icons.keyboard_arrow_right, color: Colors.grey, size: 35),
                        Space(size: 8, horizontal: true),
                        Container(
                          height: 26,
                          width: 180,
                          decoration: BoxDecoration(
                            color: Colors.white,
                            borderRadius: BorderRadius.circular(13),
                          ),
                          child: Row(
                            children: [
                              Space(size: 5, horizontal: true),
                              Icon(Icons.search, color: Colors.grey[800], size: 18),
                              Space(size: 5, horizontal: true),
                              Text('Search', style: TextStyle(color: Colors.grey[800]))
                            ],
                          ),
                        ),
                        Expanded(child: SizedBox()),
                        Container(
                          height: 30,
                          width: 30,
                          child: ClipRRect(
                            borderRadius: BorderRadius.circular(20),
                            child: Image.network(Utils.mainUser.image, fit: BoxFit.cover),
                          ),
                        ),
                        Space(size: 8, horizontal: true),
                        Text(Utils.mainUser.name, style: TextStyle(color: Colors.white)),
                        Space(size: 15, horizontal: true),
                        Icon(Icons.keyboard_arrow_down, color: Colors.grey, size: 35),
                        Space(size: 10, horizontal: true),
                      ],
                    ),
                    Expanded(child: SizedBox()),
                    Padding(
                      padding: const EdgeInsets.symmetric(horizontal: 10),
                      child: Text('Home',
                          style: TextStyle(color: Colors.white, fontWeight: FontWeight.w600, fontSize: 30)),
                    ),
                    Space(size: 15),
                  ],
                ),
              ),
            )
        ],
      ),
    );
  }
}

class ListScreen extends StatefulWidget {
  const ListScreen({Key key, this.isMobile, this.click}) : super(key: key);

  final GenericClick<SectionType> click;
  final bool isMobile;

  @override
  ListScreenState createState() => ListScreenState();
}

class ListScreenState extends State<ListScreen> {
  final _scrollController = ScrollController();
  final _threshold = 306;
  bool _playSticky = false;
  double percentage = 0.0;
  var _barOpacity = 0.0;
  var _height = 0.0;

  Playlist _playlist;

  void setPlaylist(Playlist playlist) => setState(() => _playlist = playlist);

  @override
  void initState() {
    super.initState();
    _scrollController.addListener(() {
      if (_scrollController.offset < 51) {
        _barOpacity = (_scrollController.offset - 0.0) / (50.0 - 0.0);
      } else if (_scrollController.offset > 50) {
        _barOpacity = 1.0;
      }
      if (_scrollController.offset < 180) {
        _height = (_scrollController.offset - 0.0) / (180.0 - 0.0);
      } else if (_scrollController.offset > 180) {
        _height = 1.0;
      }
      setState(() {
        percentage = _scrollController.offset / _threshold >= 1
            ? 1
            : _scrollController.offset / _threshold <= 0 ? 0 : _scrollController.offset / _threshold;
      });
      if (percentage >= 1 && !_playSticky) {
        setState(() => _playSticky = true);
      } else {
        if (percentage < 1 && _playSticky) {
          setState(() => _playSticky = false);
        }
      }
    });
  }

  @override
  void dispose() {
    _scrollController.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    final size = MediaQuery.of(context).size;
    return widget.isMobile
        ? Stack(
            children: [
              Container(
                width: size.width,
                height: size.height,
                decoration: BoxDecoration(
                  gradient: LinearGradient(
                    colors: [Color(0xFF404040), Color(0xFF181818)],
                    begin: Alignment(0.5, -1.5),
                    end: Alignment(0.5, 0.2),
                  ),
                ),
              ),
              if (_playlist != null && _playlist.element == ElementType.ARTIST)
                Container(
                  height: 380 - (percentage * 100),
                  alignment: Alignment.bottomCenter,
                  child: Stack(
                    fit: StackFit.expand,
                    children: [
                      Image.network(
                        _playlist != null && _playlist.image != null ? _playlist.image : '',
                        fit: BoxFit.cover,
                      ),
                      Container(
                        decoration: BoxDecoration(
                            gradient: LinearGradient(
                          colors: [
                            Color(0xFF181818),
                            Color(0x10181818),
                          ],
                          begin: Alignment(0.0, 1.0),
                          end: Alignment(0.0, -1.0),
                        )),
                      )
                    ],
                  ),
                ),
              Opacity(
                opacity: percentage,
                child: Container(
                  width: size.width,
                  height: size.height,
                  color: Color(0xFF181818),
                ),
              ),
              Builder(
                builder: (BuildContext context) {
                  final scale = _playlist != null && _playlist.element == ElementType.ARTIST
                      ? pow(percentage, 1.5)
                      : pow(percentage, 10);
                  final fade = 1.2 * 1 - percentage;
                  return Transform(
                    transform: Matrix4.identity()
                      ..scale(1 - (scale) < 0.8 ? 0.8 : 1 - scale, 1 - scale < 0.8 ? 0.8 : 1 - scale),
                    alignment: Alignment.bottomCenter,
                    child: Opacity(
                      opacity: fade < 0 ? 0 : fade > 1 ? 1 : fade,
                      child: Container(
                        height: 260,
                        width: size.width,
                        margin: const EdgeInsets.only(top: 60),
                        child: Column(
                          mainAxisSize: MainAxisSize.min,
                          crossAxisAlignment: CrossAxisAlignment.center,
                          mainAxisAlignment: MainAxisAlignment.end,
                          children: [
                            if (!(_playlist != null && _playlist.element == ElementType.ARTIST))
                              SizedBox(
                                height: 200,
                                width: 200,
                                child: Image.network(
                                  _playlist != null && _playlist.image != null ? _playlist.image : '',
                                  fit: BoxFit.cover,
                                ),
                              ),
                            Space(size: 20),
                            Text(
                              _playlist != null ? _playlist.name : '',
                              textAlign: TextAlign.center,
                              style: TextStyle(
                                fontWeight: _playlist != null && _playlist.element == ElementType.ARTIST
                                    ? FontWeight.w700
                                    : FontWeight.w600,
                                fontSize: _playlist != null && _playlist.element == ElementType.ARTIST ? 50 : 25,
                                color: Colors.white,
                              ),
                            )
                          ],
                        ),
                      ),
                    ),
                  );
                },
              ),
              Container(
                margin: const EdgeInsets.only(top: 50),
                child: ListView(
                  controller: _scrollController,
                  padding: const EdgeInsets.only(top: 300),
                  children: [
                    Container(
                      width: size.width,
                      height: 60,
                      child: Stack(
                        children: [
                          Align(
                            alignment: Alignment.bottomCenter,
                            child: Container(
                              color: Color(0xFF181818),
                              height: 30,
                            ),
                          ),
                          Align(
                            alignment: Alignment.bottomCenter,
                            child: Container(
                              height: 60,
                              margin: const EdgeInsets.only(bottom: 30),
                              decoration: BoxDecoration(
                                  gradient: LinearGradient(
                                colors: [
                                  Color(0xFF181818),
                                  Color(0x10181818),
                                ],
                                begin: Alignment(0.0, 1.0),
                                end: Alignment(0.0, -1.0),
                              )),
                            ),
                          ),
                          GestureDetector(
                            onTap: () {
                              widget.click(SectionType.AUDIO, (_playlist.elements as List<Song>)[0]);
                            },
                            child: Center(
                              child: FittedBox(
                                child: Container(
                                  height: 45,
                                  padding: const EdgeInsets.symmetric(horizontal: 50),
                                  decoration: BoxDecoration(
                                    color: Color(0xFF1EBA54),
                                    borderRadius: BorderRadius.circular(30),
                                  ),
                                  child: Center(
                                      child: Text(
                                          _playlist != null && _playlist.element == ElementType.ARTIST
                                              ? 'SHUFFLE PLAY'
                                              : 'PLAY',
                                          style: TextStyle(color: Colors.white, fontWeight: FontWeight.w600))),
                                ),
                              ),
                            ),
                          ),
                        ],
                      ),
                    ),
                    Container(
                      color: Color(0xFF181818),
                      width: size.width,
                      height: 40,
                      child: Row(
                        children: [
                          Space(size: 20, horizontal: true),
                          Text('Download', style: TextStyle(color: Colors.white, fontWeight: FontWeight.w600)),
                          Expanded(child: SizedBox()),
                          Switch(
                              value: false,
                              onChanged: (_) {},
                              activeColor: Colors.grey,
                              inactiveTrackColor: Colors.grey),
                          Space(size: 20, horizontal: true),
                        ],
                      ),
                    ),
                    Space(size: 10, color: Utils.sectionBg),
                    if (_playlist != null)
                      ..._playlist.elements.map((song) => SongItem(click: widget.click, song: song))
                  ],
                ),
              ),
              MobileAppBar(
                  showMenu: true,
                  title: _playlist != null ? _playlist.name : '',
                  opacity: percentage,
                  click: widget.click),
              Visibility(
                visible: _playSticky,
                child: Container(
                  margin: const EdgeInsets.only(top: 40),
                  width: size.width,
                  height: 60,
                  child: Stack(
                    children: [
                      Align(
                        alignment: Alignment.topCenter,
                        child: Container(
                          width: size.width,
                          height: 36,
                          color: Color(0xFF181818),
                        ),
                      ),
                      Align(
                        alignment: Alignment.topCenter,
                        child: FittedBox(
                          child: GestureDetector(
                            onTap: () {
                              widget.click(SectionType.AUDIO, (_playlist.elements as List<Song>)[0]);
                            },
                            child: Container(
                              height: 45,
                              margin: const EdgeInsets.only(top: 12),
                              padding: const EdgeInsets.symmetric(horizontal: 50),
                              decoration: BoxDecoration(
                                color: Color(0xFF1EBA54),
                                borderRadius: BorderRadius.circular(30),
                              ),
                              child: Center(
                                  child: Text(
                                      _playlist != null && _playlist.element == ElementType.ARTIST
                                          ? 'SHUFFLE PLAY'
                                          : 'PLAY',
                                      style: TextStyle(color: Colors.white, fontWeight: FontWeight.w600))),
                            ),
                          ),
                        ),
                      ),
                    ],
                  ),
                ),
              ),
            ],
          )
        : Stack(
            children: [
              Container(
                width: size.width,
                height: size.height,
                decoration: BoxDecoration(
                  gradient: LinearGradient(
                    colors: [Color(0xFF404040), Color(0xFF181818), Color(0xFF181818)],
                    begin: Alignment(0.5, -1.5),
                    end: Alignment(0.5, 0.2),
                  ),
                ),
                child: ListView(
                  padding: const EdgeInsets.only(top: 320, bottom: 20, left: 20, right: 20),
                  controller: _scrollController,
                  children: [
                    if (_playlist != null)
                      ..._playlist.elements.map((song) => SingleSongItem(click: widget.click, song: song))
                  ],
                ),
              ),
              Container(
                height: 300 - (_height * 180),
                color: Color(0xFF121212).withOpacity(_barOpacity > 0.6 ? 1 : _barOpacity < 0.1 ? 0 : _barOpacity),
                child: Padding(
                  padding: const EdgeInsets.symmetric(horizontal: 5),
                  child: Column(
                    crossAxisAlignment: CrossAxisAlignment.start,
                    children: [
                      Space(size: 5),
                      Row(
                        children: [
                          GestureDetector(
                            onTap: () => widget.click(SectionType.BROWSE, null),
                            child: Icon(Icons.keyboard_arrow_left, color: Colors.white, size: 35),
                          ),
                          Space(size: 5, horizontal: true),
                          Icon(Icons.keyboard_arrow_right, color: Colors.grey, size: 35),
                          Space(size: 8, horizontal: true),
                          Container(
                            height: 26,
                            width: 180,
                            decoration: BoxDecoration(
                              color: Colors.white,
                              borderRadius: BorderRadius.circular(13),
                            ),
                            child: Row(
                              children: [
                                Space(size: 5, horizontal: true),
                                Icon(Icons.search, color: Colors.grey[800], size: 18),
                                Space(size: 5, horizontal: true),
                                Text('Search', style: TextStyle(color: Colors.grey[800]))
                              ],
                            ),
                          ),
                          Expanded(child: SizedBox()),
                          Container(
                            height: 30,
                            width: 30,
                            child: ClipRRect(
                              borderRadius: BorderRadius.circular(20),
                              child: Image.network(Utils.mainUser.image, fit: BoxFit.cover),
                            ),
                          ),
                          Space(size: 8, horizontal: true),
                          Text(Utils.mainUser.name, style: TextStyle(color: Colors.white)),
                          Space(size: 15, horizontal: true),
                          Icon(Icons.keyboard_arrow_down, color: Colors.grey, size: 35),
                          Space(size: 10, horizontal: true),
                        ],
                      ),
                      Expanded(child: SizedBox()),
                      Padding(
                        padding: const EdgeInsets.symmetric(horizontal: 15),
                        child: Stack(
                          alignment: Alignment.bottomLeft,
                          children: [
                            AnimatedContainer(
                              duration: Duration(milliseconds: 250),
                              height: _barOpacity > 0.15 ? 55 : 140,
                              margin: EdgeInsets.only(bottom: _barOpacity > 0.15 ? 0 : 60),
                              child: _playlist != null
                                  ? Row(
                                      children: [
                                        AspectRatio(
                                          aspectRatio: 1,
                                          child: Image.network(
                                            _playlist.image,
                                            fit: BoxFit.cover,
                                          ),
                                        ),
                                        Space(size: 20, horizontal: true),
                                        Column(
                                          crossAxisAlignment: CrossAxisAlignment.start,
                                          mainAxisAlignment: MainAxisAlignment.center,
                                          children: [
                                            if (_barOpacity < 0.06)
                                              Text(
                                                  when(_playlist.element, {
                                                    ElementType.ARTIST: 'ENJOY THIS ARTIST',
                                                    ElementType.PLAYLIST: 'CUSTOM PLAYLIST FOR YOU'
                                                  }),
                                                  style: TextStyle(color: Colors.grey[400])),
                                            Text(_playlist.name,
                                                style: TextStyle(
                                                    color: Colors.white, fontWeight: FontWeight.w600, fontSize: 30)),
                                            if (_barOpacity < 0.06)
                                              Text(
                                                  'Made for Mariano Zorrilla - ${(_playlist.elements as List).length} song',
                                                  style: TextStyle(color: Colors.grey[400])),
                                          ],
                                        )
                                      ],
                                    )
                                  : SizedBox(),
                            ),
                            AnimatedContainer(
                              duration: Duration(milliseconds: 200),
                              curve: Curves.decelerate,
                              alignment: _barOpacity > 0.1 ? Alignment.bottomRight : Alignment.bottomLeft,
                              margin: EdgeInsets.only(bottom: _barOpacity > 0.2 ? 10 : 0),
                              child: Row(
                                crossAxisAlignment: CrossAxisAlignment.end,
                                mainAxisSize: MainAxisSize.min,
                                children: [
                                  GestureDetector(
                                    onTap: () {
                                      widget.click(SectionType.AUDIO, (_playlist.elements as List<Song>)[0]);
                                    },
                                    child: Container(
                                      height: 30,
                                      padding: const EdgeInsets.symmetric(horizontal: 45),
                                      decoration: BoxDecoration(
                                        color: Color(0xFF1EBA54),
                                        borderRadius: BorderRadius.circular(30),
                                      ),
                                      child: Center(
                                          child: Text(
                                              _playlist != null && _playlist.element == ElementType.ARTIST
                                                  ? 'SHUFFLE PLAY'
                                                  : 'PLAY',
                                              style: TextStyle(
                                                  color: Colors.white, fontWeight: FontWeight.w600, fontSize: 12))),
                                    ),
                                  ),
                                  Space(size: 10, horizontal: true),
                                  Container(
                                    height: 30,
                                    width: 30,
                                    decoration: BoxDecoration(
                                        color: Colors.black,
                                        border: Border.all(color: Colors.white, width: 1),
                                        borderRadius: BorderRadius.circular(15)),
                                    child: Icon(Icons.favorite_border, color: Colors.white, size: 14),
                                  ),
                                  Space(size: 10, horizontal: true),
                                  Container(
                                    height: 30,
                                    width: 30,
                                    decoration: BoxDecoration(
                                        color: Colors.black,
                                        border: Border.all(color: Colors.white, width: 1),
                                        borderRadius: BorderRadius.circular(15)),
                                    child: Icon(Icons.more_vert, color: Colors.white, size: 14),
                                  ),
                                ],
                              ),
                            ),
                          ],
                        ),
                      ),
                      Space(size: 15),
                    ],
                  ),
                ),
              ),
            ],
          );
  }
}

class PlayerScreen extends StatefulWidget {
  const PlayerScreen({Key key, this.song, this.audio}) : super(key: key);

  final Song song;
  final AudioElement audio;

  @override
  _PlayerScreenState createState() => _PlayerScreenState();
}

class _PlayerScreenState extends State<PlayerScreen> {
  var _isGif = false;
  var _current = 0.0;
  bool _disposed = false;
  bool _switchDesktop = true;

  @override
  void initState() {
    super.initState();
    if (mounted) {
      Future.microtask(() => setState(() => _isGif = widget.song.imgGif != null));
      Future.microtask(() {
        widget.audio.addEventListener('timeupdate', (event) {
          if (!_disposed) setState(() => _current = widget.audio.currentTime);
        }, true);
      });
    }
  }

  @override
  void dispose() {
    _disposed = true;
    super.dispose();
  }

  void handleSwitchDesktop() {
    Future.microtask(() {
      _switchDesktop = true;
      Navigator.of(context).pop();
    });
  }

  @override
  Widget build(BuildContext context) {
    final isMobile = MediaQuery.of(context).size.width < 650;
    if (isMobile) _switchDesktop = false;
    if (!_switchDesktop && !isMobile) handleSwitchDesktop();
    return Material(
      child: Container(
        decoration: BoxDecoration(
          image: _isGif ? DecorationImage(fit: BoxFit.cover, image: NetworkImage(widget.song.imgGif)) : null,
        ),
        child: Container(
          decoration: BoxDecoration(
            gradient: LinearGradient(
              colors: [
                Utils.playerTop.withAlpha(_isGif ? 0xAA : 0xFF),
                Utils.playerBottom.withAlpha(_isGif ? 0xAA : 0xFF)
              ],
              begin: Alignment.topCenter,
              end: Alignment.bottomCenter,
            ),
          ),
          child: Column(
            children: [
              MobileAppBar(
                title: widget.song.name,
                back: Icons.keyboard_arrow_down,
                onTap: () {
                  Navigator.of(context).pop();
                },
              ),
              if (!_isGif)
                Expanded(
                  child: Padding(
                    padding: const EdgeInsets.symmetric(horizontal: 65, vertical: 30),
                    child: FittedBox(
                      child: SizedBox(
                        width: 400,
                        height: 400,
                        child: Card(
                          elevation: 5,
                          shape: RoundedRectangleBorder(),
                          child: Image.network(widget.song.image, fit: BoxFit.cover),
                        ),
                      ),
                    ),
                  ),
                ),
              if (_isGif) Expanded(child: SizedBox()),
              Column(
                mainAxisSize: MainAxisSize.max,
                mainAxisAlignment: MainAxisAlignment.end,
                children: [
                  Row(
                    children: [
                      Space(size: 20, horizontal: true),
                      Column(
                        crossAxisAlignment: CrossAxisAlignment.start,
                        children: [
                          Text(widget.song.name, style: TextStyle(color: Colors.white, fontWeight: FontWeight.w600)),
                          Text(widget.song.artist.name, style: TextStyle(color: Colors.grey)),
                        ],
                      ),
                      Expanded(child: SizedBox()),
                      Icon(Icons.favorite_border, color: Colors.white),
                      Space(size: 20, horizontal: true),
                    ],
                  ),
                  Padding(
                    padding: const EdgeInsets.only(left: 20, right: 8, top: 15, bottom: 8),
                    child: SliderTheme(
                      data: SliderThemeData(
                        trackHeight: 4,
                        overlayShape: RoundSliderOverlayShape(overlayRadius: 0),
                        thumbShape: RoundSliderThumbShape(enabledThumbRadius: 6, disabledThumbRadius: 6),
                        thumbColor: Colors.white,
                        activeTrackColor: Colors.green,
                      ),
                      child: Slider(
                        value: _current <= widget.audio.duration.floor().toDouble()
                            ? _current
                            : widget.audio.duration.floor().toDouble(),
                        activeColor: Colors.white,
                        inactiveColor: Colors.grey[800],
                        max: widget.audio.duration.floor().toDouble(),
                        onChanged: (value) {
                          setState(() {
                            _current = value;
                            widget.audio.currentTime = _current;
                          });
                        },
                      ),
                    ),
                  ),
                  Row(
                    children: [
                      Space(size: 20, horizontal: true),
                      Text(Utils.parseTime(_current), style: TextStyle(color: Colors.white)),
                      Expanded(child: SizedBox()),
                      Text(Utils.parseTime(widget.audio.duration), style: TextStyle(color: Colors.white)),
                      Space(size: 20, horizontal: true),
                    ],
                  ),
                  Row(
                    children: [
                      Space(size: 20, horizontal: true),
                      Icon(Icons.shuffle, color: Colors.white),
                      Expanded(
                          child: Row(
                        mainAxisAlignment: MainAxisAlignment.center,
                        children: [
                          Icon(Icons.skip_previous, color: Colors.white, size: 30),
                          Space(size: 20, horizontal: true),
                          GestureDetector(
                            onTap: () => widget.audio.paused ? widget.audio.play() : widget.audio.pause(),
                            child: Icon(widget.audio.paused ? Icons.play_circle_filled : Icons.pause_circle_filled,
                                color: Colors.white, size: 60),
                          ),
                          Space(size: 20, horizontal: true),
                          Icon(Icons.skip_next, color: Colors.white, size: 30),
                        ],
                      )),
                      Icon(Icons.repeat, color: Colors.white),
                      Space(size: 20, horizontal: true),
                    ],
                  ),
                  Space(size: 20),
                  Row(
                    children: [
                      Space(size: 20, horizontal: true),
                      Icon(Icons.devices, color: Colors.white, size: 20),
                      Expanded(child: SizedBox()),
                      Icon(Icons.playlist_play, color: Colors.white),
                      Space(size: 20, horizontal: true),
                    ],
                  ),
                  Space(size: 20),
                ],
              )
            ],
          ),
        ),
      ),
    );
  }
}

class MobileAppBar extends StatelessWidget {
  final String title;
  final IconData back;
  final IconData icon;
  final bool showMenu;
  final double opacity;
  final GenericClick<SectionType> click;
  final GestureDragCancelCallback onTap;

  const MobileAppBar(
      {Key key,
      this.title,
      this.showMenu = true,
      this.opacity = 1.0,
      this.click,
      this.onTap,
      this.back = Icons.keyboard_backspace,
      this.icon = Icons.more_vert})
      : super(key: key);

  @override
  Widget build(BuildContext context) {
    return Container(
      height: 40,
      child: Row(
        crossAxisAlignment: CrossAxisAlignment.center,
        mainAxisSize: MainAxisSize.min,
        children: [
          Space(size: 10, horizontal: true),
          if (click != null)
            GestureDetector(
              onTap: () => click(SectionType.BROWSE, null),
              child: Icon(back, color: Colors.white),
            ),
          if (onTap != null)
            GestureDetector(
              onTap: onTap,
              child: Icon(back, color: Colors.white),
            ),
          Expanded(
            child: Align(
              alignment: Alignment.center,
              child: Opacity(
                  opacity: opacity,
                  child: Text(title ?? '', style: TextStyle(color: Colors.white, fontWeight: FontWeight.w600))),
            ),
          ),
          if (showMenu) Icon(icon, color: Colors.white),
          Space(size: 10, horizontal: true)
        ],
      ),
    );
  }
}

class PlaylistItem extends StatefulWidget {
  const PlaylistItem({Key key, this.click, this.playlist, this.small}) : super(key: key);

  final GenericClick<SectionType> click;
  final Playlist playlist;
  final bool small;

  @override
  _PlaylistItemState createState() => _PlaylistItemState();
}

class _PlaylistItemState extends State<PlaylistItem> {
  @override
  Widget build(BuildContext context) {
    final size = MediaQuery.of(context).size;
    final isMobile = MediaQuery.of(context).size.width < 600;
    return GestureDetector(
      onTap: () => widget.click(SectionType.ALBUM, widget.playlist),
      child: Container(
        width: size.width,
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Space(size: 40),
            Padding(
              padding: const EdgeInsets.only(left: 17),
              child: widget.playlist.element == ElementType.ARTIST
                  ? Row(
                      children: [
                        SizedBox(
                          width: 40,
                          height: 40,
                          child: ClipRRect(
                            borderRadius: BorderRadius.circular(20),
                            child: Image.network(widget.playlist.image, fit: BoxFit.cover),
                          ),
                        ),
                        Space(size: 10, horizontal: true),
                        Text(widget.playlist.name,
                            style: TextStyle(color: Colors.white, fontWeight: FontWeight.w600, fontSize: 20))
                      ],
                    )
                  : Text(widget.playlist.name,
                      style: TextStyle(color: Colors.white, fontWeight: FontWeight.w600, fontSize: 20)),
            ),
            Space(size: 15),
            Container(
              height: widget.small ? isMobile ? 130 : 180 : isMobile ? 200 : 250,
              child: ListView.builder(
                padding: const EdgeInsets.symmetric(horizontal: 8),
                scrollDirection: Axis.horizontal,
                itemBuilder: (context, index) {
                  final item = (widget.playlist.elements as List<Song>)[index];
                  if (item is Song) {
                    return GestureDetector(
                      onTap: () => widget.click(SectionType.AUDIO, item),
                      child: Column(
                        crossAxisAlignment: CrossAxisAlignment.start,
                        children: [
                          CoverImage(small: widget.small, item: item),
                          if (widget.small)
                            Padding(
                              padding: const EdgeInsets.only(left: 10, top: 10),
                              child:
                                  Text(item.name, style: TextStyle(color: Colors.white, fontWeight: FontWeight.w600)),
                            ),
                          if (!widget.small)
                            Padding(
                              padding: const EdgeInsets.only(left: 10, top: 10),
                              child: Column(
                                crossAxisAlignment: CrossAxisAlignment.start,
                                children: [
                                  Text(item.name, style: TextStyle(color: Colors.white, fontWeight: FontWeight.w600)),
                                  Space(size: 5),
                                  Text('${item.album.name} - ${item.artist.name}',
                                      style: TextStyle(color: Colors.grey, fontSize: 12)),
                                ],
                              ),
                            ),
                        ],
                      ),
                    );
                  } else {
                    return SizedBox();
                  }
                },
                itemCount: (widget.playlist.elements as List).length,
              ),
            ),
          ],
        ),
      ),
    );
  }
}

class CoverImage extends StatefulWidget {
  const CoverImage({Key key, this.small, this.item}) : super(key: key);

  final bool small;
  final Song item;

  @override
  _CoverImageState createState() => _CoverImageState();
}

class _CoverImageState extends State<CoverImage> {
  var _hover = false;

  @override
  Widget build(BuildContext context) {
    final isMobile = MediaQuery.of(context).size.width < 650;
    return MouseRegion(
      onHover: (_) => setState(() => _hover = true),
      onEnter: (_) => setState(() => _hover = true),
      onExit: (_) => setState(() => _hover = false),
      child: Container(
        height: widget.small ? isMobile ? 100 : 150 : isMobile ? 150 : 200,
        width: widget.small ? isMobile ? 100 : 150 : isMobile ? 150 : 200,
        margin: const EdgeInsets.symmetric(horizontal: 10),
        child: Stack(
          fit: StackFit.expand,
          children: [
            Image.network(widget.item.image, fit: BoxFit.cover),
            if (!isMobile)
              Visibility(
                visible: _hover,
                child: Container(
                  color: Colors.black.withAlpha(0xAA),
                  child: Center(
                    child: Row(
                      children: [
                        Icon(Icons.favorite_border, color: Colors.white, size: widget.small ? 18 : 22),
                        Icon(Icons.play_circle_outline, color: Colors.white, size: widget.small ? 40 : 60),
                        Icon(Icons.more_horiz, color: Colors.white, size: widget.small ? 18 : 22),
                      ],
                      mainAxisAlignment: MainAxisAlignment.spaceEvenly,
                    ),
                  ),
                ),
              )
          ],
        ),
      ),
    );
  }
}

class SingleSongItem extends StatefulWidget {
  const SingleSongItem({Key key, this.click, this.song}) : super(key: key);

  final GenericClick<SectionType> click;
  final Song song;

  @override
  _SingleSongItemState createState() => _SingleSongItemState();
}

class _SingleSongItemState extends State<SingleSongItem> {
  var _hover = false;

  @override
  Widget build(BuildContext context) {
    return GestureDetector(
      onTap: () => widget.click(SectionType.AUDIO, widget.song),
      child: MouseRegion(
        onHover: (_) => setState(() => _hover = true),
        onEnter: (_) => setState(() => _hover = true),
        onExit: (_) => setState(() => _hover = false),
        child: Container(
          color: _hover ? Colors.grey[700] : Utils.sectionBg,
          padding: const EdgeInsets.symmetric(vertical: 5),
          child: Row(
            children: [
              Space(size: 10, horizontal: true),
              Container(
                height: 30,
                width: 30,
                child: Visibility(
                  visible: _hover,
                  child: Icon(Icons.play_circle_outline, color: Colors.white, size: 30),
                ),
              ),
              Space(size: 10, horizontal: true),
              Icon(Icons.favorite_border, color: Colors.white, size: 18),
              Space(size: 15, horizontal: true),
              Expanded(
                child: Row(
                  crossAxisAlignment: CrossAxisAlignment.center,
                  children: [
                    Container(
                      child: Text(widget.song.name,
                          style: TextStyle(color: Colors.white, fontSize: 14),
                          maxLines: 1,
                          overflow: TextOverflow.ellipsis),
                      constraints: BoxConstraints(minWidth: 150, maxWidth: 250),
                    ),
                    Space(size: 20, horizontal: true),
                    Expanded(
                      child: Text(
                        '${widget.song.artist.name} - ${widget.song.album.name}',
                        style: TextStyle(fontSize: 12, color: Colors.grey),
                        overflow: TextOverflow.ellipsis,
                        maxLines: 1,
                      ),
                    ),
                  ],
                ),
              ),
              Space(size: 10, horizontal: true),
              Visibility(visible: _hover, child: Icon(Icons.more_horiz, color: Colors.grey)),
              Space(size: 10, horizontal: true),
            ],
          ),
        ),
      ),
    );
  }
}

class SongItem extends StatelessWidget {
  const SongItem({Key key, this.click, this.song}) : super(key: key);

  final GenericClick<SectionType> click;
  final Song song;

  @override
  Widget build(BuildContext context) {
    return GestureDetector(
      onTap: () => click(SectionType.AUDIO, song),
      child: Container(
        color: Utils.sectionBg,
        padding: const EdgeInsets.symmetric(vertical: 8),
        child: Row(
          children: [
            Space(size: 10, horizontal: true),
            Container(height: 50, width: 50, child: Image.network(song.image, fit: BoxFit.cover)),
            Space(size: 10, horizontal: true),
            Expanded(
              child: Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  Text(song.name, style: TextStyle(color: Colors.white, fontSize: 14)),
                  Space(size: 4),
                  Text(
                    '${song.artist.name} - ${song.album.name}',
                    style: TextStyle(fontSize: 12, color: Colors.grey),
                    overflow: TextOverflow.ellipsis,
                    maxLines: 1,
                  ),
                ],
              ),
            ),
            Space(size: 10, horizontal: true),
            Icon(Icons.more_vert, color: Colors.grey),
            Space(size: 10, horizontal: true),
          ],
        ),
      ),
    );
  }
}

class AddPlayList extends StatefulWidget {
  @override
  _AddPlayListState createState() => _AddPlayListState();
}

class _AddPlayListState extends State<AddPlayList> {
  bool _hover = false;

  @override
  Widget build(BuildContext context) {
    return MouseRegion(
      onHover: (_) => setState(() => _hover = true),
      onExit: (_) => setState(() => _hover = false),
      child: Container(
        height: 60,
        padding: const EdgeInsets.only(left: 20),
        child: Row(
          children: [
            Icon(Icons.add_circle_outline, color: _hover ? Colors.white : Colors.grey),
            Space(size: 10, horizontal: true),
            Text('New Playlist', style: TextStyle(color: _hover ? Colors.white : Colors.grey)),
          ],
        ),
      ),
    );
  }
}

class Space extends StatelessWidget {
  const Space({Key key, this.size, this.horizontal = false, this.color = Colors.transparent}) : super(key: key);

  final double size;
  final Color color;
  final bool horizontal;

  @override
  Widget build(BuildContext context) {
    return Container(width: horizontal ? size : 0, height: !horizontal ? size : 0, color: color);
  }
}

class Line extends StatelessWidget {
  const Line({Key key, this.horizontal = false, this.color}) : super(key: key);

  final bool horizontal;
  final Color color;

  @override
  Widget build(BuildContext context) {
    final size = MediaQuery.of(context).size;
    return Container(
        height: horizontal ? 1 : size.height, width: horizontal ? size.width : 0.5, color: color ?? Colors.grey[800]);
  }
}

enum ElementType { PLAYLIST, ARTIST, ALBUM, SONG }
enum TextType { HEADER, NORMAL, TITLE, BOLD, SPACE }
enum SectionType {
  HOME,
  BROWSE,
  BROWSE_PODCAST,
  PODCAST,
  GRID,
  GRID_CIRCLE,
  GRID_SECTION,
  LIST,
  LIST_IMAGE,
  ARTIST,
  ALBUM,
  SONG,
  AUDIO,
  FULLSCREEN
}

class Utils {
  static const Color leftPanelBg = Color(0xFF121212);
  static const Color rightPanelBg = Color(0xFF121212);

  static const Color sectionBg = Color(0xFF181818);

  static const Color playerSectionBg = Color(0xFF282828);

  static const Color selectedItem = Color(0xFF20D760);

  static const Color playerTop = Color(0xFF964847);
  static const Color playerBottom = Color(0xFF191317);

  static List<User> users = [
    User(
      'Martin Aguinis',
      'https://pbs.twimg.com/profile_images/1139005628846878721/lSg5Loq4_400x400.jpg',
      'Summer Nights',
      'LiQWYD',
      '2h',
    ),
    User(
      'Tim Sneath',
      'https://pbs.twimg.com/profile_images/653618067084218368/XlQA-oRl.jpg',
      'Atch',
      'Your Daily Mix 1',
      '10h',
    ),
    User(
      'Simon',
      'https://pbs.twimg.com/profile_images/1017532253394624513/LgFqlJ4U_400x400.jpg',
      'Dawn',
      'MusicbyAden',
      '21h',
    ),
  ];

  static User mainUser = User('Mariano Zorrilla',
      'https://pbs.twimg.com/profile_images/1222274976415281153/TVSI4DIx_400x400.jpg', null, null, null);

  static List<Item> leftSectionItems = [
    Item('Home', true, TextType.HEADER, SectionType.HOME, GlobalKey<LeftListItemState>()),
    Item('Browse', false, TextType.HEADER, SectionType.BROWSE, GlobalKey<LeftListItemState>()),
    Item('Radio', false, TextType.HEADER, SectionType.GRID, GlobalKey<LeftListItemState>()),
    Item('YOUR LIBRARY', false, TextType.TITLE, null, GlobalKey<LeftListItemState>()),
    Item('Made for you', false, TextType.BOLD, SectionType.GRID_SECTION, GlobalKey<LeftListItemState>()),
    Item('Recently Played', false, TextType.BOLD, SectionType.GRID, GlobalKey<LeftListItemState>()),
    Item('Liked Song', false, TextType.BOLD, SectionType.LIST, GlobalKey<LeftListItemState>()),
    Item('Albums', false, TextType.BOLD, SectionType.GRID, GlobalKey<LeftListItemState>()),
    Item('Artists', false, TextType.BOLD, SectionType.GRID_CIRCLE, GlobalKey<LeftListItemState>()),
    Item('Podcasts', false, TextType.BOLD, SectionType.PODCAST, GlobalKey<LeftListItemState>()),
    Item(null, false, TextType.SPACE, null, null),
    Item('PLAYLISTS', false, TextType.TITLE, SectionType.LIST, GlobalKey<LeftListItemState>()),
    Item('Discover Weekly', false, TextType.NORMAL, SectionType.LIST, GlobalKey<LeftListItemState>()),
    Item('Starred', false, TextType.NORMAL, SectionType.LIST, GlobalKey<LeftListItemState>()),
  ];

  static String parseTime(num time) {
    var seconds = time % 60;
    var foo = time - seconds;
    var minutes = foo / 60;
    var formatSeconds = seconds.toInt().toString();
    if (seconds < 10) {
      formatSeconds = '0' + formatSeconds;
    }
    return '$minutes:$formatSeconds';
  }

  static void itemSelection<T extends List>(T list, dynamic selected) {
    list.asMap().forEach((key, value) {
      if (list[key].key != null && list[key].key.currentState != null) {
        list[key].key.currentState.selected(selected is bool ? selected : key == selected);
      }
    });
  }

  static List<Playlist> homeList = [
    Playlist('Recently Played', 'https://assets.codepen.io/2399829/daily.png', ElementType.PLAYLIST, [
      atchVoyage,
      memoriesMusicByAden,
      coffee,
      virtualJoySilentCrafter,
      summerNights,
      boraBora,
      youngLove,
      intense,
      overSoon,
      sakura,
      soul,
      followMe,
      callMe,
    ]),
    Playlist('LiQWYD', 'https://assets.codepen.io/2399829/liqwyd.jpg', ElementType.ARTIST, [
      overSoon,
      summerNights,
      callMe,
      coffee,
      soul,
      youngLove,
      fadeOut,
      eyesClosed,
      flow,
      feelIt,
      breathe,
      leap,
    ]),
    Playlist('Dance & Electronic', 'https://assets.codepen.io/2399829/edm.jpg', ElementType.PLAYLIST, [
      soul,
      laser,
      youngLove,
      boraBora,
      memoriesMusicByAden,
      takeAwayThePain,
      intense,
      eyesClosed,
      callMe,
    ]),
    Playlist('Vendredi', 'https://assets.codepen.io/2399829/vendredi.jpg', ElementType.ARTIST, [
      takeAwayThePain,
      followMe,
      tropicalLove,
      ai,
      cocktail,
      rooftop,
      humAndThrill,
      silhouette,
      trueStory,
      mistyFantasy,
      firework,
      divingIn,
    ]),
  ];

  static Artist get atch => Artist('Atch', 'https://i1.sndcdn.com/avatars-000724049929-uwg00o-t500x500.jpg', []);

  static Album get atchAlbum =>
      Album('Voyage', 'https://i1.sndcdn.com/artworks-04xcPMCHF5woz843-CwdjVQ-t500x500.jpg', atch, []);

  static Song get atchVoyage => Song(
        'Voyage',
        'https://assets.codepen.io/2399829/arch_voyage.jpg',
        null,
        'https://assets.codepen.io/2399829/voyage_atch.mp3',
        atch,
        atchAlbum,
      );

  static Song get memoriesMusicByAden => Song(
        'Memories',
        'https://i1.sndcdn.com/artworks-OSGuqLsjDoSnpGZN-ihwdsA-t500x500.jpg',
        null,
        'https://assets.codepen.io/2399829/memories_musicbyaden.mp3',
        musicByAden,
        memoriesAlbum,
      );

  static Album get memoriesAlbum => Album('MusicByAden', getThumbnail('4fX1VmXg-gU'), musicByAden, []);

  static Artist get musicByAden => Artist('MusicByAden', getThumbnail('4fX1VmXg-gU'), []);

  static Song get virtualJoySilentCrafter => Song(
        'Virtual Joy',
        'https://assets.codepen.io/2399829/virtualjoy.jpg',
        'https://assets.codepen.io/2399829/virtualjoy.gif',
        'https://assets.codepen.io/2399829/virtual_joy_silentcrafter.mp3',
        silentCrafter,
        silentCrafterHappy,
      );

  static Song get laser => Song(
        'Laser',
        'https://assets.codepen.io/2399829/laser_silentcrafter.jpg',
        null,
        'https://assets.codepen.io/2399829/laser_silentcrafter.mp3',
        silentCrafter,
        silentCrafterHappy,
      );

  static Album get silentCrafterHappy => Album('Happy', getThumbnail('eNgusegQBGM'), silentCrafter, []);

  static Artist get silentCrafter => Artist('SilentCrafter ', getThumbnail('eNgusegQBGM'), []);

  static Song get boraBora => Song(
        'Bora Bora',
        'https://assets.codepen.io/2399829/bora.jpg',
        null,
        'https://assets.codepen.io/2399829/bora_bora_mbb.mp3',
        mbb,
        calmAlbum,
      );

  static Album get calmAlbum => Album('Calm', getThumbnail('Qvw-s27xp_g'), mbb, []);

  static Artist get mbb => Artist('MBB ', getThumbnail('Qvw-s27xp_g'), []);

  static Song get intense => Song(
        'Intense',
        'https://assets.codepen.io/2399829/intense.jpg',
        null,
        'https://assets.codepen.io/2399829/intense_peyruis.mp3',
        peyruis,
        bright,
      );

  static Album get bright => Album('Bright', getThumbnail('AOjdw8XX-vE'), peyruis, []);

  static Artist get peyruis => Artist('Peyruis ', getThumbnail('AOjdw8XX-vE'), []);

  static Song get overSoon => Song(
        'Over Soon',
        getThumbnail('FfQ5aULtfTY'),
        null,
        'https://assets.codepen.io/2399829/over_soon_liqwyd.mp3',
        liQWYD,
        chill,
      );

  static Song get summerNights => Song(
        'Summer Nights',
        'https://assets.codepen.io/2399829/summer_nights.jpg',
        'https://assets.codepen.io/2399829/summer_nights.gif',
        'https://assets.codepen.io/2399829/summer_nights_liqwyd.mp3',
        liQWYD,
        chill,
      );

  static Song get callMe => Song(
        'Call Me',
        'https://assets.codepen.io/2399829/callme.jpg',
        null,
        'https://assets.codepen.io/2399829/call_me_liqwyd.mp3',
        liQWYD,
        chill,
      );

  static Song get coffee => Song(
        'Coffee',
        'https://assets.codepen.io/2399829/coffee_liqwyd.jpg',
        null,
        'https://assets.codepen.io/2399829/coffee_liqwyd.mp3',
        liQWYD,
        chill,
      );

  static Song get soul => Song(
        'Soul',
        getThumbnail('n6soxrKiCos'),
        null,
        'https://assets.codepen.io/2399829/soul_liqwyd.mp3',
        liQWYD,
        chill,
      );

  static Song get youngLove => Song(
        'Young Love',
        'https://assets.codepen.io/2399829/young_love_liqwyd.jpg',
        null,
        'https://assets.codepen.io/2399829/young_love_liqwyd.mp3',
        liQWYD,
        chill,
      );

  static Song get fadeOut => Song(
        'Fade Out',
        getThumbnail('64AGaW9GuWM'),
        null,
        'https://assets.codepen.io/2399829/fade_out_liqwyd.mp3',
        liQWYD,
        chill,
      );

  static Song get eyesClosed => Song(
        'Eyes Closed',
        getThumbnail('O4jNueFacQM'),
        null,
        'https://assets.codepen.io/2399829/eyes_closed_liqwyd.mp3',
        liQWYD,
        chill,
      );

  static Song get flow => Song(
        'Flow',
        getThumbnail('A52eKafrihU'),
        null,
        'https://assets.codepen.io/2399829/flow_liqwyd.mp3',
        liQWYD,
        chill,
      );

  static Song get feelIt => Song(
        'Feel It',
        getThumbnail('mTczsFoP2Cc'),
        null,
        'https://assets.codepen.io/2399829/feel_it_liqwyd.mp3',
        liQWYD,
        chill,
      );

  static Song get breathe => Song(
        'Breathe',
        getThumbnail('UzOXNwtkcYs'),
        null,
        'https://assets.codepen.io/2399829/breathe_liqwyd.mp3',
        liQWYD,
        chill,
      );

  static Song get leap => Song(
        'Leap',
        getThumbnail('9DhAPuKtqV8'),
        null,
        'https://assets.codepen.io/2399829/leap_liqwyd.mp3',
        liQWYD,
        chill,
      );

  static Album get chill => Album('Chill', 'https://assets.codepen.io/2399829/album_liqwyd.jpg', liQWYD, []);

  static Artist get liQWYD => Artist('LiQWYD', 'https://assets.codepen.io/2399829/liqwyd.jpg', []);

  static Song get sakura => Song(
        'Sakura 2020',
        'https://assets.codepen.io/2399829/sakura.png',
        'https://assets.codepen.io/2399829/sakura.gif',
        'https://assets.codepen.io/2399829/sakura_2020_roa.mp3',
        roa,
        roaAlbum,
      );

  static Album get roaAlbum => Album('Roa', 'https://assets.codepen.io/2399829/roa_music.jpg', roa, []);

  static Artist get roa => Artist('Roa', 'https://assets.codepen.io/2399829/roa_music.jpg', []);

  static Song get followMe => Song(
        'Follow Me',
        'https://assets.codepen.io/2399829/follow_me_vendredi.jpg',
        null,
        'https://assets.codepen.io/2399829/follow_me_vendredi.mp3',
        vendredi,
        vendrediAlbum,
      );

  static Song get tropicalLove => Song(
        'Tropical Love',
        getThumbnail('8C-9VIKe-VQ'),
        'https://assets.codepen.io/2399829/tropical_love.gif',
        'https://assets.codepen.io/2399829/tropical_love_vendredi.mp3',
        vendredi,
        vendrediAlbum,
      );

  static Song get ai => Song(
        'AI',
        getThumbnail('-TRUcjFes4M'),
        null,
        'https://assets.codepen.io/2399829/ai_vendredi.mp3',
        vendredi,
        vendrediAlbum,
      );

  static Song get cocktail => Song(
        'Cocktail',
        getThumbnail('0GVbYKhVaxI'),
        null,
        'https://assets.codepen.io/2399829/cocktail_vendredi.mp3',
        vendredi,
        vendrediAlbum,
      );

  static Song get rooftop => Song(
        'Rooftop Marrakech',
        getThumbnail('A4GNhPf-YYE'),
        null,
        'https://assets.codepen.io/2399829/rooftop_marrakech_vendredi.mp3',
        vendredi,
        vendrediAlbum,
      );

  static Song get humAndThrill => Song(
        'Hum & Thrill',
        getThumbnail('7tZyeBLrAJ4'),
        null,
        'https://assets.codepen.io/2399829/hum_and_thrill_vendredi.mp3',
        vendredi,
        vendrediAlbum,
      );

  static Song get silhouette => Song(
        'Silhouette',
        getThumbnail('WhqnxycuUXQ'),
        null,
        'https://assets.codepen.io/2399829/silhouette_vendredi.mp3',
        vendredi,
        vendrediAlbum,
      );

  static Song get trueStory => Song(
        'True Story',
        getThumbnail('sgKMJQWKkk4'),
        null,
        'https://assets.codepen.io/2399829/true_story_vendredi.mp3',
        vendredi,
        vendrediAlbum,
      );

  static Song get mistyFantasy => Song(
        'Misty Fantasy',
        getThumbnail('fazCGsYP938'),
        null,
        'https://assets.codepen.io/2399829/misty_fantasy_vendredi.mp3',
        vendredi,
        vendrediAlbum,
      );

  static Song get firework => Song(
        'Firework',
        getThumbnail('r5W-txT2GWQ'),
        null,
        'https://assets.codepen.io/2399829/firework_vendredi.mp3',
        vendredi,
        vendrediAlbum,
      );

  static Song get divingIn => Song(
        'Diving In',
        getThumbnail('6hHkS95U65E'),
        null,
        'https://assets.codepen.io/2399829/diving_in_vendredi.mp3',
        vendredi,
        vendrediAlbum,
      );

  static Song get takeAwayThePain => Song(
        'Take The Pain Away',
        getThumbnail('1p0KYaAGMC4'),
        null,
        'https://assets.codepen.io/2399829/take_the_pain_away_vendredi.mp3',
        vendredi,
        vendrediAlbum,
      );

  static Album get vendrediAlbum => Album('Vendredi', 'https://assets.codepen.io/2399829/vendredi.jpg', vendredi, []);

  static Artist get vendredi => Artist('Vendredi', 'https://assets.codepen.io/2399829/vendredi.jpg', []);

  static Song currentSong;

  static String getThumbnail(String url) => 'http://i3.ytimg.com/vi/$url/maxresdefault.jpg';

  static Tuple<Widget, AudioElement> audio(String url, id, bool autoPlay) {
    final audio = AudioElement();
    audio.autoplay = autoPlay;
    audio.src = url;
    // ignore:undefined_prefixed_name
    ui.platformViewRegistry.registerViewFactory('audio_$url$id', (int viewId) => audio);
    return Tuple(HtmlElementView(key: UniqueKey(), viewType: 'audio_$url$id'), audio);
  }
}

class Item {
  String text;
  bool selected;
  TextType type;
  SectionType section;
  GlobalKey<LeftListItemState> key;

  Item(this.text, this.selected, this.type, this.section, this.key);
}

class Playlist {
  String name;
  String image;
  ElementType element;
  dynamic elements;

  Playlist(this.name, this.image, this.element, this.elements);
}

class Artist extends Element {
  String name;
  String image;
  List<Album> albums;

  Artist(this.name, this.image, this.albums);
}

class Album extends Element {
  String name;
  String image;
  Artist artist;
  List<Song> songs;

  Album(this.name, this.image, this.artist, this.songs);
}

class Song extends Element {
  String name;
  String image;
  String imgGif;
  String url;
  Artist artist;
  Album album;

  Song(this.name, this.image, this.imgGif, this.url, this.artist, this.album);
}

class User {
  String name;
  String image;
  String song;
  String playlist;
  String time;

  User(this.name, this.image, this.song, this.playlist, this.time);
}

class Src {
  String src;

  Src(this.src);
}

class Element {}

typedef ItemClick = Function(int index);
typedef GenericClick<T> = Function(T section, dynamic item);

class Tuple<F, S> {
  F first;
  S second;

  Tuple(this.first, this.second);
}

Type when<Input, Type>(Input selectedOption, Map<Input, Type> branches, [Type defaultValue]) {
  if (!branches.containsKey(selectedOption)) {
    return defaultValue;
  }
  return branches[selectedOption];
}
