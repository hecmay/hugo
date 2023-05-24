---
title: 'AuTool: porting browser extension to desktop'
date: 2023-05-02T15:05:55-04:00
draft: false
tags: ['plugin system', 'DSL', 'automation']
---

## Quick demo
{{< youtube 8FOwCdPLD0c >}}

## Features

### Write plugins in YAML
Every AuTool plugin is a YAML file that defines various actions, such as mouse clicks, keyboard inputs, web interactions, and more.

```yaml
actions:
    - window.in(System Settings):
        # Mouse click relative to the window
        - mouse.click({{ [0,200] }})
```

### Interact with popup windows
If your plugin needs to obtain user inputs to determine what actions to perform, you can display a popup window that requests input from the user using `user.input` API. 

```yaml
actions:
    # Make a handy GUI for command-line tools or REST APIs, making them more user-friendly.
    - >-
      user.input({{ {
        'title': 'Example',
        'content': {
          'type': 'text',
          'label': 'Enter user name',
          'key': 'username'
        }
      } }}) => $resp
    
    # Use the user input to run CLI commands
    - os.shell( echo {{ $resp['username'] }})
```

### Guide people to use softwares
You can use `window.annotate` API to draw floating bounding boxes or shapes on software windows. These annotations provide detailed notes when the cursor is hovered over them, making it easier to guide users on how to use the software.

```yaml
actions:
    - window.is(System Settings):
        - >- 
          window.annotate({{ [
            {
              'type': 'box',
              'position': [0, 0],
              'size': [100, 100]
            } 
          ] }})
```

### Connect apps into a pipeline
You can connect multiple apps into a pipeline. It supports both desktop apps and web apps. 

```yaml
actions:
  # send a discord message when new email arrives
  - event.on(__IMAP__, {{ {'email':'...', 'key':'...'} }}) => $r:
      - data.read( {{ $r['title'] }}) => $title
      - web.http(POST, https://discord.com/api/webhooks/...)

```


## Example Plugins
### AI assistants
- GPT chatbot
- Self-hosted chatbot built on LLaMA and Vicuna
- GPT-based summarizer for local articles
- Image generation with stable diffusion
- Image super-resolution
- Image segmentation
- Image background removal
- Audio-to-Caption realtime converter

### Easy cloud service
- Retailer price monitoring and auto-checkout
- RSS or website status monitoring service
- Conditionally distribute your message to other users
- Tracing service (e.g., tracing QR code scans, or email opens)

### Software collaboration
- Send a poll to your team members and trigger a task based on the response
- Notify your team on Slack when your social media account is mentioned
- Post a task in Trello when a new email with specific keywords is received
- Save a screenshot to Google Drive

### Other desktop applets
- Video and image format converter
- App and file launcher
- Shortcut to fav web pages (e.g., Memos)
- QR code generator

## GUI components
### Elements

- Text input or display-only text label
```yaml
    { 
      'type': 'text',
      'label': 'Enter your name',

      # it is display-only if `key` is missing
      'key': 'name'
    }
```

- Select from a list of static options

```yaml
    { 
      'type': 'select',
      'label': 'Select your favorite options',
      'options': ['option1', 'option2', 'option3'],

      # it is display-only if `key` is missing
      'key': 'selected'
      'max': 5,
      'min': 1
    }
```

- Select from a list of dynamic options. This can be useful when you want to search local disk, call REST APIs, or run a command to get a list of options dynamically.

```yaml
    { 
      'type': 'dynamic',
      'label': 'Select your favorite options',

      # this can be a URL or a command that returns a list
      'source': 'wss://localhost:5678/files',
      'params': {
        'path': '/home/user/desktop'
        'callback': 'https://localhost:5678/'
      },
    }
```

### Layouts
- List: arrange elements in vertical order

```yaml
    { 
      'type': 'list',
      'content': [
        {...}, {...}
      ]
    }
```

- Tabs: arrange elements in different pages/tabs

```yaml
    { 
      'type': 'tabs',
      'content': [
        {...}, {...}
      ]
    }
```

## Why did I write this tool?

- Make it easier to accommodate to some custom automation needs, and avoid installing so many apps just for a simple task. Especially when some apps are heavyweight but you only need a small feature in it.

- Give user control to define and adjust the GUI according to their own needs. You can easily build a handy and minimal GUI tool to call your favorite command line tools or REST APIs.

- Provide a structured way to define GUI and automation, so that it can be easily learned by LLMs, and used to generate automation pipelines with full human control.
  

### What's next?
- IR and program generation for robots, something like [GPT for robotics](https://www.microsoft.com/en-us/research/group/autonomous-systems-group-robotics/articles/chatgpt-for-robotics/). The basic idea is to provide an ISA like interface for robots, and use GPT to generate programs for robots. 


demo: talk and control a GPT-powered robot.
{{< youtube PgT8tPChbqc >}}

- Integrate with IoT devices. There is a lot of potential interesting applications, such as geo-location based games (e.g., when you enter a specific area, you will be prompted with some notifications by someone), controlling smart home devices, etc.

e.g., a portable, modular, and extensible computer

{{< youtube b3F9OtH2Xx4 >}}