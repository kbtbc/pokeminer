# pokeminer+

[![Build Status](https://travis-ci.org/Noctem/pokeminer.svg?branch=develop)](https://travis-ci.org/Noctem/pokeminer)

A Pokémon Go scraper capable of scanning large areas for Pokémon spawns over long periods of time. Features spawnpoint scanning, Twitter and PushBullet notifications, accurate expiration times and estimates based on historical data, pokestop and gym collection, a CAPTCHA solving script, and more.

[A demonstration of the Twitter notifications can be viewed here](https://twitter.com/SLCPokemon).


## How does it work?

It uses a database table of spawnpoints and expiration times to visit points soon after Pokemon spawn. For each point it determines which eligible worker can reach the point with the lowest speed, or tries again if all workers would be over the configurable speed limit. This method scans very efficiently and finds Pokemon very soon after they spawn, and also leads to unpredictable worker movements that look less robotic. The spawnpoint database continually expands as Pokemon are discovered. If you don't have enough accounts to keep up with the number of spawns in your database, it will automatically skip points that are unreachable within the speed limit or points that it has already seen spawn that cycle from other nearby points.

If you don't have an existing database of spawn points it will spread your workers out over the area you specify in config and collect the locations of spawn points from GetMapObjects requests. It will then visit those points whenever it doesn't have a known spawn (with its expiration time) to visit soon. So it will gradually learn the expiration times of more and more spawn points as you use it.

There's also a simple interface that displays active Pokemon on a map, and can generate nice-looking reports.

Here it is in action:

![in action](pokeminer/static/demo/map.png)

Since it uses [Leaflet](http://leafletjs.com/) for mapping, the appearance and data source can easily be configured to match [any of these](https://leaflet-extras.github.io/leaflet-providers/preview/) with the `MAP_PROVIDER_URL` config option.

## Features

- accurate timestamp information whenever possible with historical data
- Twitter and PushBullet notifications
  - references nearest landmark from your own list
- IV/moves detection, storage, and notification
  - produces nice image of Pokémon with stats for Twitter
  - can configure to get IVs for all Pokémon or only those eligible for notification
- stores Pokémon, gyms, and pokestops in database
- spawnpoint scanning with or without an existing database of spawns
- automatic account swapping for CAPTCHAs and other problems
- pickle storage to improve speed and reduce database queries
- manual CAPTCHA solving that instantly puts accounts back in rotation
- closely emulates the client to reduce CAPTCHAs and bans
- automatic device_info generation and retention
- aims at being very stable for long-term runs
- able to map entire city (or larger area) in real time
- reports for gathered data
- asyncio coroutines
- support for Bossland's hashing server
  - displays key usage stats in real time

## Setting up
1. Install Python 3.5 or later (3.6 is recommended)
2. `git clone https://github.com/Noctem/pokeminer.git` or download the [zip](https://github.com/Noctem/pokeminer/archive/develop.zip)
3. Copy `config.example.py` to `pokeminer/config.py` and customize it with your accounts, location, database information, and any other relevant settings. The comments in the config example provide some information about the options.
4. `pip3 install -r requirements.txt`
  * Optionally `pip3 install` additional packages listed in optional-requirements
    * *pushbullet.py* is required for pushbullet notifications
    * *python-twitter* is required for twitter notifications
    * *stem* is required for proxy circuit swapping
    * *shapely* is required for landmarks or spawnpoint scan boundaries
    * *selenium* (and [ChromeDriver](https://sites.google.com/a/chromium.org/chromedriver/)) are required for solving CAPTCHAs
    * *uvloop* provides better event loop performance
    * *pycairo* is required for generating IV/move images
    * *mysqlclient* is required for using a MySQL database
    * *psycopg2* is required for using a PostgreSQL database
    * *requests* is required for using webhooks
    * *aiosocks* is required for using SOCKS proxies
    * *cchardet* and *aiodns* provide better performance with aiohttp
    * *numba* provides better performance through JIT compilation
5. Run `python3 scripts/create_db.py` from the command line
6. Run `python3 scan.py`
  * Optionally run the live map interface and reporting system: `python3 web.py`


**Note**: Pokeminer works with Python 3.5 or later only. Python 2.7 is **not supported** and is not compatible at all since I moved from threads to coroutines. Seriously, it's 2016, Python 2.7 hasn't been developed for 6 years, why don't you upgrade already?

Note that if you want more than 10 workers simultaneously running, SQLite is likely not the best choice. I personally use and recommend PostgreSQL, but MySQL and SQLite should also work.


## Reports

There are three reports, all available as web pages on the same server as the live map:

1. Overall report, available at `/report`
2. Single species report, available at `/report/<pokemon_id>`
3. Gym statistics page, available by running `gyms.py`

The workers' live locations and stats can be viewed from the main map by enabling the workers layer, or at `/workers` (communicates directly with the worker process and requires no DB queries).

Here's what the overall report looks like:

[![report](https://i.imgur.com/LH8S85dm.jpg)](pokeminer/static/demo/report.png)

The gyms statistics server is in a separate file, because it's intended to be shared publicly as a webpage.

[![gyms](https://i.imgur.com/MWpHAEWm.jpg)](pokeminer/static/demo/gyms.png)

## Getting Started Tips & FAQs

**PokeMiner Upgrading**

The `develop` branch changes almost daily.  To you ensure you are on the most recent `develop` build, from your pokeminer directory:
```
git checkout develop
git pull
pip3 install -U -r requirements.txt
```

**Q. How can I install the optional python dependencies on Windows (not available through pip3)?**

See http://www.lfd.uci.edu/~gohlke/pythonlibs/ for most of the files.  Use `pip3 install [filename]` . Uvloop is not supported on Windows.

**Q. What mapping tools can I use to create my polygon boundaries?** 

Here are a couple that will work:
* http://geojson.io/  (note long/lat is reversed in the json) -- good for saving and later editing
* https://www.keene.edu/campus/maps/tool/

**Q: What is the format to use for my accounts in config.py if they all have same passwords/provider?**

`ACCOUNTS = [['x'], ['y']]`

**Q. What do the dots represent in the status display?**

Dots meaning:
```        
        . = visited more than a minute ago
        , = visited less than a minute ago, no pokemon seen
        0 = visited less than a minute ago, no pokemon or forts seen
        : = visited less than a minute ago, pokemon seen
        ! = currently visiting
        | = cleaning bag
        $ = spinning a PokéStop
        * = sending a notification
        ~ = encountering a Pokémon
        I = initial, haven't done anything yet
        ^ = waiting to log in (limited by SIMULTANEOUS_LOGINS)
        ∞ = bootstrapping
        L = logging in
        A = simulating app startup
        T = completing the tutorial
        X = something bad happened
        H = waiting for the next period on the hashing server
        C = CAPTCHA
```
Other letters: various errors and procedures

**Q. How long will it take to find 100% TTH?**

Depends mostly on the number of workers for the spawnpoint density you have (varies widely).  It will be however long it takes for a worker to go to the same point often enough to get the last 90s of the spawn.  The last 90s is the only time when a valid TTH is sent by Niantic to the client.

**Q. Can I use multiple hash keys?**

You can add multiple hash key support by changing HASH_KEY to a tuple of keys and changing this line in worker.py:

from: `self.api.activate_hash_server(config.HASH_KEY)`
to: `self.api.activate_hash_server(choice(config.HASH_KEY))`

Note: pgoapi's mechanism for keeping track of maximum and remaining hashes only applies to the most recently used key, so you'll have a little weirdness with those stats jumping between keys.

**Q. Can I run multiple instances of Pokeminer?**

Yes, for now the easiest thing is to copy the contents (except pickle files) to a separate directory.  In Windows, you will also need to define a unique `MANAGER_ADDRESS` in config.py for each instance.

**Q. What are the differences between known/unknown count in console display vs. map display?**

The console display is based on pickle data stored inside the cell ids.  The web.py map uses the database.   Known are spawn points that it knows the cycle for, unknown/mysteries are spawn points that it doesn't know the cycle for. GetMapObjects returns a list of spawn points for S2 cells. Pokeminer uses those so that it has places to go when it hasn't seen enough pokemon to have many points.  

**Q. What does the `--no-pickle` flag do?**

This mode tells Pokeminer to skip loading cell points from the pickle files (and use the database instead).  If you disable MORE_POINTS, it is a good idea to use `--no-pickle`.  It is also a good idea to use `--no-pickle` if you change or constrict your scan area since there will likely be mystery points from the old area in spawns.pickle.      Cell (MORE) points don't show up on the map (they are loaded from the database).

**Q. What does the `--bootstrap` flag do?**

This mode spreads out the workers evenly over your grid and repeats visits to those initial points if something goes wrong. It does that automatically if no spawn points are known (like on the first run).

**Q. What is the difference between GIVE_UP & SKIP_SPAWN?**

SKIP is whether it tries to find a worker or not.

GIVE_UP is how long it tries to find a worker.

Consider these settings:
```
GIVE_UP_KNOWN = 75   # try to find a worker for a known spawn for this many seconds before giving up
GIVE_UP_UNKNOWN = 60 # try to find a worker for an unknown point for this many seconds before giving up
SKIP_SPAWN = 90      # don't even try to find a worker for a spawn if the spawn time was more than this many seconds ago
```
If a pokemon spawned 90s ago and no worker was able to go there in that 90s it will skip.  It skips that point if it spawned more than 90 seconds ago.

So with SKIP_TIME at 90, it's iterating through the spawn points and it sees that the spawn point spawned 91 seconds ago, it skips it without even trying to find a worker. The next one spawned 89 seconds ago so it starts trying to find a worker, if there aren't any workers that can get to it under the speed limit it waits a couple seconds and tries again until GIVE_UP seconds have passed since the time it started trying.

If you have many more spawn points than you can keep up with, you can either be visiting them 10 minutes after they spawn and have 20 minutes of warning, or you can be visiting them 90 seconds after and have 28.5 minutes of warning.
If you can keep up with your spawn points you'll probably have workers ready before 90 seconds have passed.


**Q: How can I use notification filters to send *everything* to the webhooks?**

The easiest way:
```
NOTIFY_IDS = None
NOTIFY_RANKING = None
ALWAYS_NOTIFY_IDS = tuple(range(1, 152))
```
But, it would be faster and smarter (and use less hashing) to enable some filtering in Pokeminer so that it can skip sending stuff to the webhook that will be ignored anyways.  One option would be putting everything in NOTIFY_IDS and overriding their rarity scores.

For example:
```
NOTIFY_IDS = (149, 143, 19, 16)
RARITY_OVERRIDE = {149: 1, 143: .7, 19: 0.2, 16:0.2}
INITIAL_SCORE = 0.6
MINIMUM_SCORE = 0.6
```
Here they all need to have a total score of at least 0.6. Dragonite has a rarity score of 1, so it only needs a .2 to hit the threshold. Pidgey has a rarity score of 0.2 so it would need perfect IVs to hit it.

Note:  if the rarity scores weren't overridden in that example, they would be generated based on the order listed, so 1, .66, .33, 0.

This should also accomplish similar:
```
ALWAYS_NOTIFY_IDS = (3, 6, 9, 143, 149)
INITIAL_SCORE = 0.9
MINIMUM_SCORE = 0.9
NOTIFY_IDS = tuple(range(1, 152))
IGNORE_RARITY = True
```




## License

See [LICENSE](LICENSE).

This project is based on the coroutines branch of (now discontinued) [pokeminer](https://github.com/modrzew/pokeminer/tree/coroutines). Pokeminer was originally based on an early version of [PokemonGo-Map](https://github.com/AHAAAAAAA/PokemonGo-Map), but no longer shares any code with it. It currently uses my lightly modified fork of [pgoapi](https://github.com/Noctem/pgoapi).
