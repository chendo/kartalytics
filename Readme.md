<div align="center">
  <h1>😬🏁<br>Kartalytics.</h1>
  <h3>Kart effortlessly. Cut constantly.</h3>
</div>

# Kartalytics

Kartalytics is a project to in real time analyse Mario Kart 8 Deluxe matches as they're being played and then take action on that analysis.

It is comprised of 3 main parts:

- [Recorder](#recorder)
- [Analyser](#analyser)
- [Brain](#brain)

### Scope

Kartalytics is designed to handle only 3 and 4 player VS matches at 150cc (what we play competitively).

## Recorder

Its function is to capture the input stream of Mario Kart and save it out as sequence of snapshots. To do this we'll use the following hardware:
  - HDMI splitter
  - HDMI to ethernet converter ([LKV373A](http://www.ebay.com.au/itm/LKV373A-V3-0-HDMI-Extender-100-120M-HDMI-Extender-Over-Cat5-Cat6-TCP-IP-based-/162287794299))
  - Raspberry Pi (but any 'nix) would be fine.

### Running

To setup the recorder:

  1. Install ffmpeg
  2. Thanks to [Danman](https://blog.danman.eu/new-version-of-lenkeng-hdmi-over-ip-extender-lkv373a/) you need to block 0 byte UDP packets that the encoder spits out - use this command: `sudo iptables -t raw -A PREROUTING -p udp -m length --length 28 -j DROP`
  3. `> cd dump && ffmpeg -skip_frame nokey -i "udp://239.255.42.42:5004?localaddr=169.254.244.97&buffer_size=128000&overrun_nonfatal=1&fifo_size=500000" -vf fps=5 -qscale:v 5 out%04d.jpg`

You might need to change the IP addresses - my RaspberryPi v3 can maintain a 2fps (500ms) sample rate - faster computer you could increase the `fps=` sample more images.

## Analyser

Its function is to take the raw snapshot images and analyse them to determine what is happening in the image.  It is stateless and it simply populate an event stream with events such as:
  - Load screen
  - Main Menu
  - Race Start (race_name)
  - Race Underway
    - Player 1 (position: 6, items: [red shell, none], coins: 5)
    - etc
  - Race finish:
    - Player 1 (position: 2, points: 12)
  - View Results:
    - Player 1 (position: 3, points: 87)

A complete race could look something like this:

```json
[
  {"event_type":"loading_screen","timestamp":"2017-07-02T12:09:16.273Z"},
  {"event_type":"intro_screen","data":{"course_name":"Big Blue"},"timestamp":"2017-07-02T12:09:25.783Z"},
  {"event_type":"race_screen","data":{"player_one":{"position":7}},"timestamp":"2017-07-02T12:09:38.273Z"},
  {"event_type":"race_screen","data":{"player_two":{"position":4},"player_three":{"position":3},"player_four":{"position":2}},"timestamp":"2017-07-02T12:09:39.273Z"},
  {"event_type":"race_screen","data":{"player_one":{"position":5},"player_two":{"position":4}},"timestamp":"2017-07-02T12:09:41.773Z"},
  {"event_type":"race_screen","data":{"player_one":{"position":5},"player_two":{"position":4},"player_three":{"position":3},"player_four":{"position":7}},"timestamp":"2017-07-02T12:09:42.273Z"},
  {"event_type":"race_result_screen","data":{"player_one":{"position":1}},"timestamp":"2017-07-02T12:12:01.293Z"},
  {"event_type":"race_result_screen","data":{"player_one":{"position":1},"player_two":{"position":5},"player_three":{"position":8},"player_four":{"position":9}},"timestamp":"2017-07-02T12:12:01.813Z"},
  {"event_type":"race_result_screen","data":{"player_one":{"position":1},"player_two":{"position":5},"player_four":{"position":9}},"timestamp":"2017-07-02T12:12:03.793Z"},
  {"event_type":"match_result_screen","timestamp":"2017-07-02T13:00:26.038Z","data":{"player_two":{"position":5},"player_three":{"position":8},"player_four":{"position":9}}},
  {"event_type":"match_result_screen","timestamp":"2017-07-02T13:00:26.038Z","data":{"player_one":{"position":1},"player_two":{"position":5},"player_three":{"position":8},"player_four":{"position":9}}}
]
```

Notice that the `match_result_screen` gets possibly incomplete data - animations cause this.

### Running

```
> bundle
> POST_URL=http://your-brain-location/new_events ruby daemon.rb
```

For debugging purposes you can also set `KEEP_FILES=true` - this will instuct the daemon not to remove processed files.

## Brain (not included)

The brain is responsible for consuming the event stream and reconstituting those into a concept of Games/Players/Results etc.  As an example, the brain sees a series of race underway events then a "Main Menu" event, it should assume the game has been abandoned.  Likewise if it sees a series of Race Underway events then a Race Finish then a View Results it should assume a game has been completed.

Hopefully I can open source this shortly.
