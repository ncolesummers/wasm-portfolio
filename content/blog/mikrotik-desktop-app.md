
# This is the preamble, and it is written in TOML format.
# In this section, you set information about the page, like title, description, and the template
# that should be used to render the content.

# REQUIRED

# The title of the document
title = "Building a Configuration Generator for Mikrotiks Using Wails and React"

# OPTIONAL

# The description of the page.
description = "contaiNERD and WASM dev"

# The name of the template to use. `templates/` is automatically prepended, and `.hbs` is appended.
# So if you set this to `blog`, it becomes `templates/blog.hbs`.
# template = ""

# These fields are user-definable. You can create whatever values you want
# here. The format must be `string` keys with `string` values, though.
[extra]
date = "Apr. 15, 2021"

# Anything after this line is considered Markdown content
# Building a Configuration Generator for Mikrotiks Using Wails and React
I work at an Internet Service Provider (ISP) in Idaho, and we use [Mikrotiks](https://mikrotik.com) extensively for fiber and fixed wireless deployments.  Configuring routers by hand was a common cause of errors on our network.  The high number of unique configurations drastically increased the troubleshooting complexity of support calls.

This project began as a simple python script to automate the on-premises configuration of every Mikrotik that we deploy.  The project requirements kept increasing until the terminal application became unwieldy and confusing.  The script's primary users have little terminal experience, so it was not the best medium for them.  The wireless technicians are frequently in areas without internet or cellular reception, so a web app would not be possible.  I started development of a simple desktop application that replaced the terminal application.

Switching to the compiled language "Go" proved to be an easy migration from the scripting and compiling with pyinstaller in the old terminal solution.   And, learning to code in Go was not difficult. Go's templating library made generating the configurations a breeze.  Using the new "embed" package allowed including all of my templates directly into the final binary. Here's an example template that adds a DHCP Client to ether1: 

```jinja2
{{define "dhcpClient"}}
### DHCP Client ###
{
/ip dhcp-client add interface=ether1 use-peer-dns=yes add-default-route=yes dhcp-options=hostname,clientid disabled=no
/log info message=“DHCP client Configured”
}
{{- end}}
```

Before attempting to build a solution in "Wails," I created a GUI using [Fyne](https://fyne.io).  Fyne was easy to build with, and I could make all of the desktop components using Go.  Unfortunately, The legacy laptops I have to support don't have a recent graphics driver that would work with OpenGL, so I had to find another solution.  [Wails](https://wails.app) is that solution.  Wails is a cross-platform desktop application framework that uses a web-view and web technologies to create a User Interface (UI).  Now I can use [React](https://reactjs.org), the most popular framework for building UIs, and not rely on Go's fledgling GUI  support.  The fact that Wails uses mshtml, a win32 API that hasn't seen an update since Internet Explorer version 11 (IE11), was a feature in my case.

Building with Wails is as simple as binding a function to the front-end:

```go
app.Bind(builder.BuildRouter)
```

And calling the function using Javascript/Typescript.  In the below example, I'm passing a Javascript object, which gets converted to `map[string]interface{}` on the Go side.

```js
var myRouter = {
	Username: this.state.username,
  Password: this.state.password,
  Installation: this.state.selectedInstall,
  DisableWiFi: this.state.disableWiFi,
  SSID: this.state.ssid,
  WPA2: this.state.wpa2,
  Bridge: this.state.bridged,
  LTE: false
}

window.backend.BuildRouter(myRouter)
```

After passing the map to the backend, it is converted to a struct, as shown below.

```go
  var router model.Router

  err := mapstructure.Decode(data, &router)
  if err != nil {
    log.Println(“Error decoding map to struct: “, err)
  }
```

The struct is then passed to the corresponding template to be executed, where it writes the executed template to the clipboard and a file in the current directory.  

The UI is a simple form with two radio groups for selections, a row of checkboxes, and three to five input fields for the installer to fill in, depending on the choices selected.  Each field has validation, and the generate button is enabled when all of the visible fields are valid. Clicking the generate button passes the form data to the backend, as shown above.  The installer then pastes the configuration file into the Mikrotik through a terminal and lets the configuration script work its magic.       

[image:C44D1B5F-D167-4D04-BF2C-759F3CBC5E44-13711-0000011B9339E81E/mikrotik-config-scrot.png]

Wails made it simple to create a front-end for my application. I'd be willing to use it again for desktop development.  Although, for future projects, I am considering another path using [Tauri](https://tauri.studio). 