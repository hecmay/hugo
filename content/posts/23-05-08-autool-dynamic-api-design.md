---
title: "GUI dynamic API design"
date: 2023-05-01T15:05:55-04:00
draft: false
tags: ["autool", "plugins"]
---

## API for dynamic types
- No callback: the default behavior is to add all selected items into an editable list.

```yaml
{ 
  "type": "dynamic", 
  "label": "Search file and click to open", 

  "options": {
      "source": "wss://localhost:5678",
      "params": { "type": "Files", "path": "/home/user" },
      "instantQuit": True
  }
}
```

- Case 1: process a selected file and return the results in stream (e.g., GPT summarized PDF content). The returned results maybe in streaming mode, so we probably need 

```yaml
{ 
  "type": "dynamic", 
  "label": "Search files and click to summarize", 

  "options": {
      "source": "wss://localhost:5678",
      "params": { "type": "Files", "path": "/home/user" },

      # upload file to server when item is selected
      "onSelect":
        {
          "source": "https://192.168.0.30:8080/summarize",
          "params": { 
            "maxLength": 100
          },
        },
    }
}
```

- Case 2: search through command line or REST API returns. A quick example is to use todo.sh, and return the pending TODO items as data table (a long table)


## API for dynamic tables
- Showing a data table for editing list of data, which can be useful in TODO list, or editing list of people (e.g., attending an events or something)


## Other optimizations
- **Modular rendering**: separate the DOM rendering functions into smaller modules, and use global store to sync data between them.

- **Modular backend requests**: separate the backend requests into smaller modules, and use global store to sync data between them.

- **Web service provider**: provide a web service to (1) authorize users, e.g., machine metadata, subscription, (2) amazon

- **Updating the content in existing session**: use `session_id` to identify the session, and use `session_id` to update the content in the existing session. 

```yaml
actions:
  - data.read({{ [] }}) => $selected

  - event.on(__MOUSE_CLICKED__) => $location:
    # Get a screen shot of the current active window
    - window.capture($location) => $image

    # Send to server for image segmentation
    - web.http(POST, $image) => $segmentation

    # Annotate the image with segmentation results
    - window.annotate($segmentation)

    # append the annotated image to the selected list
    - data.read( {{ }} ) => $selected
    
    - user.input($selected) => $selected

  - cmd.sleep(1800s)

```

- **Interaction session**: (low priority) build a Moon OS like terminal emulator in AuTool, so that users can try out the APIs in a terminal-like environment.