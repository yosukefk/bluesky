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

