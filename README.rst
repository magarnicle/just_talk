Author
------

Matthew Rademaker

matthew@acctv.com.au

Purpose
-------
To provide a python model of the Tools On Air Broadcast API for use with the Tools On Air suite of programs.

https://toolsonair.atlassian.net/wiki/spaces/TST/pages/81100962/Broadcast+Suite+Communication+Protocol

Installation
------------

.. code-block:: bash
    $ pip install just_talk

To perform a refresh of a playlist requires the MediaInfo dll

??HOW DOES ONE INSTALL THIS??

Usage
-----

To update a day in Just:Play, first connect to the channel by calling comms.connect() with your
channels details.

Then call messages.request_node() with the date of the day you wish to update. The date must be
a model.toaTimecode, which can be constructed like a datetime e.g. model.toaTimecode(2017, 12, 19).

You can now manipulate this day object, adding or removing playlists and any sub-nodes like tracks
and plays.

Once you have finished, call update() on the day object and the updated day will be sent to Just:Play.
The three pieces of functionality that make your life easier are:

 1. Child nodes can be accessed as a list e.g. a_day.playlists, a_playlist.video_tracks
 2. Adding a Playlist to a Day, a Track to a Playlist, or a Play to a Track will timeout EVERYTHING
    in the model - it will search up the tree as far as it can and time everything out. Slow, yes,
    but do you want to do it yourself? Do you want it to NOT be timed out properly?
 3. Changing the start or duration of a play or playlist will automatically calculate a field called
    'end'

While it is possible to update individual playlists etc., it may be easier to only ever update whole days.
This is because time and subsequently TV doesn't really operate in days; it is a continuous flow of video.
Humans, and subsequently Just:Play, DOES think in days. So if you insert a video into a day that pushes the
total video time of a day to greater than 24 hours, Just:Play does not gracefully handle this. Long story
short, it gets very messy. Just update whole days at a time.

N.B. I do not know what will happen if you update the current day while it is playing. It may go well for you;
you are welcome to try it on your live channel if you need your fix of adrenaline for the day.

Example
=======

The following will add a video to your channel. !!!WARNING This will remove all current playlists for tomorrow!!!
.. code-block:: python
    import just_talk as jt
    from datetime import datetime, timedelta

    details = jt.connect(<IP of your channel>, <Port of your channel>, <Username for your channel>, <Password for your channel (if needed)>)
    print details
    tomorrow_dt = datetime.now() + timedelta(days=1)
    # Clear the day
    tomorrow = jt.request_node(tomorrow_dt)
    jt.request_delete(tomorrow.playlists)
    # Refresh our copy of the day, since it has changed in TOA
    tomorrow = jt.request_node(tomorrow_dt)
    # Easy setup of a blank day with a new video
    tomorrow.setup('<path to a video>')
    tomorrow.update()
    jt.disconnect()

The above code uses the Day.setup() function, which is a good example of how to manipulate the model:
.. code-block:: python

    def setup(self, video):
        if not self.playlists:
            # A playlist has a track, which has a play. They all need an id.
            pl = Playlist(nid=1)
            t = VideoTrack(nid=1)
            p = Play(nid=1)
            p.resource = (0, video)
            t.plays.append(p)
            pl.video_tracks.append(t)
            # This will get all the details of the video from the file itself
            pl.refresh(auto_update=False)
            # If the playlist doesn't start on the same day as the day we are
            # adding it to it will throw an error
            pl.toaStart = self.toaStart
            self.playlists.append(pl)


Design
------

The just_talk package has low-level communications for connecting to the playout system (comms.py),
functions to call the various messages (messages.py) and a model of all the data types and objects
used by the playout system, specifically Just:Play (model.py).

Most of the complexity is in just_talk.model. Just:Talk objects (days, playlists etc.) are classes,
all deriving from a 'Node', which in turn is an xml object, since Just:Talk uses xml for its playlists.

One quirk of the implementation (which may be fixed in the future) is that only data stored in the actual
underlying xml will persist. For example, if you have a Day object, and do the following:
.. code-block:: python
    a_day.playlists[0].new_field = 'test'
    
Then try this:
.. code-block:: python
    print a_day.playlists[0].new_field
    
The result will be blank. This is because all gets and sets are interacting with the underlying xml,
NOT with the python object. The attributes of an object in the model are created each time they are
accessed from the xml itself, and the xml is created from a template file stored in the xml folder
in the just_talk package. These xml files correspond to the Just:Talk implementation, and only
fields in there will be accepted by Just:Talk, so it is probably a good thing that new fields can't
be added.

You can add fields to the current object, and they'll still be there for you.
So, this will work:
.. code-block:: python
    a_day.new_field = 'test'
    print a_day

Asyncronous messaging
=====================

Because I fell asleep in my networking courses, the messaging between just_talk and TOA is not
100% reliable. Sometimes it will only catch part of the return messages. It's a determined little
package, though, so it'll keep trying to get the whole message for a while before giving up.
This means that the same message will be sent a number of times, but this doesn't affect the result.
However, if anyone out there sees the code in comms.py and thinks "what muppet wrote this?", his name is
Matt and he'd love to know how to write it properly.
