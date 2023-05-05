---
title: 'AuTool: porting browser extension to desktop'
date: 2023-05-02T15:05:55-04:00
draft: false
tags: ['plugin system', 'DSL', 'automation']
---

## Quick demo
{{< youtube 8FOwCdPLD0c >}}

## Features
### Write plugins in yaml
Specify automation process in yaml, such as mouse, keyboard, web, visual (e.g., image recognition), and many other types of actions. 

```yaml
actions:
    # click a location in target window
    - window.is(System Settings):
        - mouse.click({{ [0,200] }})
```

### Simple GUI for plugins using JSON
You can write a simple plugin with GUI for your favorite command line tools or REST APIs, in just a few lines of code.

```yaml
actions:
    # This will trigger a popup window to ask for inputs
    - >-
      user.input({{ {
        'title': 'Input your name'
        'content': {
          'type': 'text',
          'label': 'Enter your name'
        },
      } }}) => $resp
```

### Screen annotations
Inside plugins, you can use screen annotations to highlight a region, or draw a shape on the screen, to guide users to use the software.

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
You can connect multiple apps into a pipeline, and control the flow of the pipeline using a simple GUI.

```yaml
actions:
  # send a discord message when new email arrives
  - event.on(__IMAP__, {{ {'email':'...', 'key':'...'} }}) => $r:
      - data.read( {{ $r['title'] }}) => $title
      - web.http(POST, https://discord.com/api/webhooks/...)

```

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