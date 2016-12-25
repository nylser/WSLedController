Address(localnet-specific): alarmpi

Primary Port: 8765

Paths:

/get Protocol

	-> expects UUID, enters into dict
	sends back current color
	every 1u: sends current color

/set Protocol

	-> expects entered UUID
	if mode = constant color:
		expects rgb colours; format (r, g, b), eg. (0, 0, 0)
	if mode = fading-mode:
		expects rgb,rgb colours; format (r1, g1, b1, r2, g2, b2)

/modeset Protocol:
	expects: "const" for constant-color mode; "fade" for color-fade mode; changes mode accordingly, if not already set.


Server implementations:

WSLedController(python) (Multiple client support)

Client implementations:

WebController (works with JS & colorpicker)

