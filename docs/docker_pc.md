## Using Docker on PC to run BlueSky

Docker command line works more smoothly on bash/Linux.  On windows, PowerShell would make most sense to run docker from command line.  Some syntax needs to be adjusted to do that.


Running image without a commond, this works as is.

```powershell
docker run --rm bluesky
```

Piped input example.

* line continuation for bash (\\) is backtick (\`) for PowerShell
* unless you installed it there is no `less` on PC.  Use `more`

```powershell
echo '{
"fires": [
    {
	"id": "SF11C14225236095807750",
	"event_of": {
	    "id": "SF11E826544",
	    "name": "Natural Fire near Snoqualmie Pass, WA"
	},
	"activity": [
	    {
		"active_areas": [
		    {
			"start": "2015-01-20T17:00:00",
			"end": "2015-01-21T17:00:00",
			"ecoregion": "southern",
			"utc_offset": "-09:00",
			"perimeter": {
			    "geometry": {
				"type": "Polygon",
				"coordinates": [
				    [
					[-121.4522115, 47.4316976],
					[-121.3990506, 47.4316976],
					[-121.3990506, 47.4099293],
					[-121.4522115, 47.4099293],
					[-121.4522115, 47.4316976]
				    ]
				]
			    }
			}
		    }
		]
	    }
	]
    }
]
}' | docker run --rm -i bluesky `
bsp fuelbeds ecoregion consumption emissions --indent 4 | more
```

run with input file, here we need trick. PowerShell has environmental variables like $pwd, but it has Windows path, and docker has hard time interpreting it.  so we have to parse it to like bash path.

```powershell
# Function to convert windows path to what docker likes
filter docker-path {'/' + $_ -replace '\\', '/' -replace ':', ''}
```

with docker-path function defined, command looks like below

```powershell
docker run --rm -ti --user bluesky `
    -v ( ( echo $HOME/code/pnwairfire-bluesky/ | docker-path ) + ':/bluesky/' ) `
    bluesky bsp `
    -i /bluesky/dev/data/json/2-fires-24hr-20140530-CA.json `
    fuelbeds ecoregion consumption emissions
```

So the change is `-v $HOME/code/pnwairfire-bluesky/:/bluesky/` for docker on linux to `-v ( ( echo $HOME/code/pnwairefire=bluresky/ | docker-path ) + ':/bluesky/' )` .  it uses two pairs of parenthesis.  Original `-v` command has path on host machine, `$HOME/code/pnwairfire-bluesky/` and path wihin docker container `/bluesky/`, joined by colon `:`.   Inner parenthesis parse `$HOME` part to be compatible with docker `( echo $HOME/code/pnwairfire-bluesky/ | docker-path )` .  then the results is treated as string, and joined with remaining portion of the argument `:/bluesky/`, with plue operator.  needs single quote to treat the path as string.

Very ugly there would be better way but the commdna above works.  very time you want to use `-v` command, Windows side path needs to be translated with docker-path as in this example, and then joined with `+ ':/absolute/path/in/container'`

### Using image for development

[docker.md](docker.md) says

    To use the docker image to run bluesky from your local repo, you need to
    set PYTHONPATH and PATH variables in your docker run command.  

that would be like this
```powershell
docker run --rm -ti --user bluesky `
    -v ( ( echo $HOME/code/pnwairfire-bluesky/ | docker-path ) + ':/bluesky/' ) `
    -e PYTHONPATH=/bluesky/ `
    -e PATH=/bluesky/bin/:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin `
    bluesky bsp -h
```

But i am not quire sure why that's needed.  Moreover, if we run python script on Windows machine (-v mounted) from docker, it has issue of text file format ("\n" for linux, "\r\n" for pc).  if we really need to have this distinction of mounted drive, work around for PC vs Linux script is needed.  for 

So, instead i propose following command, ignoreing -e options to set path.  I
also dropped `-ti` flag, that is meant to use commadn interactively.  that's
doesnt make sense for `bsp` command, as there is no interactive use for the
command.  i dont see the point of `--user` option so dropped it.  And tailing
slash in path is redundant (i think), so i am dropping them for the rest of examples.

```powershell
docker run --rm `
    -v ( ( echo $HOME/code/pnwairfire-bluesky | docker-path ) + ':/bluesky' ) `
    bluesky bsp -h
```

Similarly for the next command would be like that.   Note that `-o` option
arguments are enclosed in single quote, because {run_id} appears to be parsed
by PowerShell, which is not what we want.


```powershell
docker run --rm `
    -v ( ( echo $HOME/code/pnwairfire-bluesky | docker-path ) + ':/bluesky' ) 
    -w /bluesky `
    bluesky `
    bsp --log-level=DEBUG --indent 4 `
    -i ./dev/data/json/2-fires-24hr-20140530-CA.json `
    -o './output/{run_id}.json' `
    fuelbeds ecoregion consumption emissions
```

Next is example with vsmoke to estimate dispersion

```powershell
docker run --rm `
    -v ( ( echo $HOME/code/pnwairfire-bluesky | docker-path ) + ':/bluesky' ) `
    -w /bluesky `
    bluesky `
    bsp --log-level=DEBUG --indent 4 `
    --run-id 'vsmoke-1-fire-72-hr-{timestamp:%Y%m%dT%H%M%S}' `
    -i ./dev/data/json/1-fire-72hr-20140530-CA.json `
    -o './output/{run_id}.json' `
    -c ./dev/config/dispersion/dispersion-vsmoke-72hr.json `
    fuelbeds ecoregion consumption emissions timeprofile dispersion
```

Next two are example with Hysplit.  Not confirmed for success, as I couldn't figure out how to feed met file for hysplit on my machine. `findmet` modeule fails.

```powershell
docker run --rm `
    -v ( ( echo $HOME/code/pnwairfire-bluesky | docker-path ) + ':/bluesky' ) `
    -v ( ( echo $HOME/Met/CANSAC/4km/ARL | docker-path ) + ':/data/Met/CANSAC/4km/ARL' ) `
    -w /bluesky `
    bluesky `
    bsp --log-level=DEBUG --indent 4 `
    --run-id 'hysplit-{timestamp:%Y%m%dT%H%M%S}' `
    -i ./dev/data/json/1-fire-24hr-20190610-CA.json `
    -o './output/{run_id}.json' `
    -c ./dev/config/fuelbeds-through-visualization/DRI4km-2019061012-48hr-PM2.5-grid-latlng.json `
    fuelbeds ecoregion consumption emissions `
    timeprofile findmetdata localmet plumerise `
    dispersion visualization
```

```powershell
docker run --rm `
    -v ( ( echo $HOME/code/pnwairfire-bluesky | docker-path ) + ':/bluesky' ) `
    -v ( ( echo $HOME/Met/PNW/4km/ARL | docker-path ) + ':/data/Met/PNW/4km/ARL' ) `
    -w /bluesky `
    bluesky `
    bsp --log-level=DEBUG --indent 4 `
    --run-id '2-fires-24hr-2019-07-26-WA-{timestamp:%Y%m%dT%H%M%S}' `
    -i dev/data/json/2-fires-24hr-2019-07-26-WA.json `
    -o './output/{run_id}.json' `
    -c ./dev/config/fuelbeds-through-visualization/PNW4km-2019072600-24hr-PM2.5.json `
    fuelbeds ecoregion consumption emissions `
    timeprofile findmetdata localmet plumerise `
    dispersion visualization
```

#### Docker wrapper script

This most likely need editing to deal with windows path

#### Running docker in interactive mode

Below gives bash prompt from inside the the container.  

```powershell
docker run --rm -ti `
    -v ( ( echo $HOME/code/pnwairfire-bluesky | docker-path ) + ':/bluesky' ) `
    -v ( ( echo $HOME/Met/NAM/12km/ARL | docker-path ) + ':/data/Met/NAM/12km/ARL' )`
    -w /bluesky `
    bluesky bash
```

Example tells to run command like below.


    ./bin/bsp --log-level=DEBUG \
        -i ./dev/data/json/2-fires-24hr-20140530-CA.json \
        -c ./dev/config/fuelbeds-through-visualization/DRI6km-2014053000-24hr-PM2.5-compute-grid-km.json \
        fuelbeds ecoregion consumption emissions timeprofile \
        findmetdata localmet plumerise dispersion \
        visualization export --indent 4 > out.json

But above doesnt work because (1) there is no ./bin, whose aboslute path would be /bluesky/bin (2) there is no "./dev/.../DRI6km- ... -compute-grid-km.json" file.  there is "DRI6km-2014053000-24hr-PM2.5-user-defined-grid-km.json' in the same directory.  Not sure these work similarly or i am missing step to create what's in the example.   Also, findmet module is not working for me.  in any case, code below would work if you have figured out this met file dieal.

```bash
bsp --log-level=DEBUG \
    -i ./dev/data/json/2-fires-24hr-20140530-CA.json \
    -c ./dev/config/fuelbeds-through-visualization/DRI6km-2014053000-24hr-PM2.5-user-defined-grid-km.json \
    fuelbeds ecoregion consumption emissions timeprofile \
    findmetdata localmet plumerise dispersion \
    visualization export --indent 4 > out.json
```

### Notes about using Docker

#### Mounted volumes
#### Cleanup

These notes in [docker.md](docker.md) applies to dockers on PC as well.

### Running other tools in docker

#### BlueSkyKml
#### BlueSky Output Visualizer
#### Other tools

Haven't tested yet

