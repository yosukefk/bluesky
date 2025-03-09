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

Next example is here

```powershell
docker run --rm -ti --user bluesky `
    -v ( ( echo $HOME/code/pnwairfire-bluesky/ | docker-path ) + ':/bluesky/' ) `
    -e PYTHONPATH=/bluesky/ `
    -e PATH=/bluesky/bin/:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin `
    bluesky bsp -h
```

```powershell
docker run --rm  -ti --user bluesky `
    -v (( echo $HOME/code/pnwairfire-bluesky/ | docker-path ) + ':/bluesky/' ) 
    -e PYTHONPATH=/bluesky/ `
    -e PATH=/bluesky/bin/:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin `
    -w /bluesky/ `
    bluesky `
    bsp --log-level=DEBUG --indent 4 `
    -i ./dev/data/json/2-fires-24hr-20140530-CA.json `
    -o './output/{run_id}.json' `
    fuelbeds ecoregion consumption emissions
```

-ti --user -e will be dropped for the rest

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

```powershell
docker run --rm `
    -v ( ( echo $HOME/code/pnwairfire-bluesky | docker-path ) + ':/bluesky' ) `
    -v ( ( echo  $HOME/Met/CANSAC/4km/ARL | docker-path ) + ':/data/Met/CANSAC/4km/ARL' ) `
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

```powershell
docker run --rm -ti `
    -v ( ( echo $HOME/code/pnwairfire-bluesky | docker-path ) + ':/bluesky' ) `
    -v ( ( $HOME/Met/NAM/12km/ARL | docker-path ) + ':/data/Met/NAM/12km/ARL' )`
    -w /bluesky `
    bluesky bash
```
