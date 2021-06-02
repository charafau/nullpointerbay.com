---
title: "Flutter Linux Plugin Development Setup"
date: 2021-06-03T00:07:31+01:00
cover:
    image: "/flutter_linux_plugin.png"
---

This post was also posted to [Flutter Community Medium](https://medium.com/flutter-community/flutter-linux-plugin-development-setup-e30f504baadd)

Thanks to Flutter support for the Linux desktop, we can now expect more beautiful apps for this great platform. However, when you want to write a custom plugin to expand your app's functionality, you need to write some C code. I haven’t found instructions on how to do that anywhere, so here is how to set up Visual Studio Code to develop and debug Flutter plugins for Linux.

# Before you start

I expect that you already have Linux build tools and VS Code installed, but if not, here are two links on how to do that:

https://flutter.dev/desktop#additional-linux-requirements

https://code.visualstudio.com/

# Create your plugin from template

First, you need to create your project. Assuming you have Flutter in your *PATH* variable, invoke this command:

`flutter create - org com.example - platforms=linux myapp`

Where *myapp* is the name of the plugin. (You can also add more platforms after a comma).

# Compile project first

On Linux, Flutter uses CMake's tool when compiling embedder code and launches GTK window with our app. To debug native code, first, you need to run a flutter project. It will make a tool to create necessary symlinks for cmake project. Inside *myapp/example*, run our favorite command:

`flutter run`

After that, quit the app.

# Opening main project

Now the open root of our newly created plugin ( *myapp/* ) inside VS Code, next installs Microsoft C++ plugin - https://marketplace.visualstudio.com/items?itemName=ms-vscode.cpptools . You will be asked to join the Insider channel for the plugin; you can join it if you want new features to land faster.

![CMake integration](/flutter_plugin_01.png#center)

![Flutter uses a Clang compiler, so choose Clang.](/flutter_plugin_02.png#center)

You will also be asked if you want CMake integration for the project, choose Yes, and next click *Locate Cmake* file.

![Choose "Locate" and navigate to the file.](/flutter_plugin_03.png#center)

You will be asked for the location of the CMakeList.txt file and make you choose the one inside:

`myapp/example/linux/CMakeLists.txt`

Do not choose the one inside myapp/Linux, or IntelliSense will not work.

![Location of CMakeLists.txt](/flutter_plugin_04.png#center)

Now relaunch VS Code.

At this point, code completion should work.

# Setting up debugging

Debugging with print statements is not the best way to analyze your code so let's configure a proper debugger.


![Location of setting files](/flutter_plugin_05.png#center)

The inside root folder ( *myapp/* ) create subfolder *.vscode* (notice the dot) and inside it a file named *c_cpp_properties.json*  - that's where vscode's C++ plugin reads config from:

```
{
  "configurations": [
    {
      "name": "Linux",
      "includePath": [
        "${workspaceFolder}/**"
      ],
      "defines": [],
      "compilerPath": "/usr/bin/clang",
      "cStandard": "c17",
      "cppStandard": "c++14",
      "intelliSenseMode": "linux-clang-x64",
      "compileCommands": "${workspaceFolder}/build/compile_commands.json",
      "configurationProvider": "ms-vscode.cmake-tools"
    }
  ],
  "version": 4
}
```

You also will need a settings.json file for cmake config:


```
{
  "cmake.sourceDirectory": "${workspaceFolder}/example/linux",
  "cmake.configureOnOpen": true,
  "files.associations": {
    "cstring": "cpp"
  }
}
```

Finally, create `launch.json` with two launch configurations inside it:

```
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Launch Flutter",
      "cwd": "example",
      "request": "launch",
      "type": "dart"
    },
    {
      "name": "Debug native",
      "type": "cppdbg",
      "request": "launch",
      "program": "${workspaceFolder}/example/build/linux/x64/debug/bundle/myapp_example",
      "cwd": "${workspaceFolder}"
    }
  ]
}
```

You need to adjust the name of our executable created by the flutter run command. Take a look at this part of JSON:

```
{
  "program": "${workspaceFolder}/example/build/linux/x64/debug/bundle/myapp_example"
}
```

The name of our executable is myapp_example. It might be different if you created a project with a different name. If you're running a different version of Flutter, then the path might be slightly different. You can check if that's the right binary by simply running it.

# Running the code

We created two launch configurations, one for running flutter with a dart, like the usual, and the second one for the running created binary with C++ debugger.

One caveat to this approach is that you will have to rerun flutter run if you change something in dart code.

![Great success!](/flutter_plugin_06.png#center)

Is it perfect? No, but is it better than printing variables to console? ***YES***

# Closing

Hope you find this useful and hopefully I will see more Flutter plugin on Linux shortly.

Happy Coding!