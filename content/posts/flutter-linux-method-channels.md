---
title: "Using Flutter Method Channels on Linux "
date: 2021-09-22T00:07:31+01:00
cover:
    image: "/channel.jpg"
---


Photo by [Pau Sayrol](https://unsplash.com/@pausayrol?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)





Hello again! Today I will continue my Flutter Linux journey and we will touch Linux integration again.  [Last time]({{< relref "posts/flutter-linux-plugin-development-setup.md" >}}) we managed to setup Visual Studio Code for plugin development, today we will take a closer look at [Method Channels](https://flutter.dev/docs/development/platform-integration/platform-channels). 

If you want to create a plugin with Method Channel support it's quite easy, just generate a Flutter project from template using `flutter create -t plugin --platforms=linux <name of your project>`. However, what if you don't want to create a separate plugin and just add some custom code to your Flutter Linux application? 
I've found out it's not straightforward so I thought that I will write this article so you won't have to figure it out by yourself.

To not prolong it any longer, let's just get started.


# Create method channel 

First we need to create `codec`, `binary messenger` and `channel`. Next we assign a method call back to our custom method which we will create in the next step.

To do so let's open `my_application.cc` inside `linux` folder and navigate to 
`my_application_activate` function. Next we implement objects described above
after plugins initialization.

In the example below, `name_of_our_channel` is the method which we are calling from Dart code.


```
static void my_application_activate(GApplication *application)
{

  ...
  // Other standard code for our application

  fl_register_plugins(FL_PLUGIN_REGISTRY(view));

  // START OF OUR CUSTOM  BLOCK

  // Get engine from view
  FlEngine *engine = fl_view_get_engine(view);

  g_autoptr(FlStandardMethodCodec) codec = fl_standard_method_codec_new();
  g_autoptr(FlBinaryMessenger) messenger = fl_engine_get_binary_messenger(engine);
  g_autoptr(FlMethodChannel) channel =
      fl_method_channel_new(messenger,
                            "name_of_our_channel",  // this is our channel name
                            FL_METHOD_CODEC(codec));
  fl_method_channel_set_method_call_handler(channel, 
  // Method which will be called when we call invokeMethod() from dart
                                            method_call_cb,  
                                            g_object_ref(view),
                                            g_object_unref);

  fl_method_channel_set_method_call_handler(channel, 
                            method_call_cb, g_object_ref(view), g_object_unref);

  // END OF OUR CUSTOM BLOCK

  gtk_widget_grab_focus(GTK_WIDGET(view));
}
```

# Callback function

Now let's create callback function:


```
static void method_call_cb(FlMethodChannel *channel,
                           FlMethodCall *method_call,
                           gpointer user_data) {}
```

Pretty straightforward, we pass a channel, methodcall and some user data.

To check for the channel's method name we need to use the `fl_method_call_get_name` function on `method_call` object. And compare it with `strcmp` like so:


```
static void method_call_cb(FlMethodChannel *channel,
                           FlMethodCall *method_call,
                           gpointer user_data)
{
  const gchar *method = fl_method_call_get_name(method_call);
  if (strcmp(method, "my_method") == 0)
  {
      // Code for "my_method"
  }
}
```

### Method not implemented response

If method passed to channel does not exist, we need to return not implemented result to do this we need to call `fl_method_call_respond`



```
static void method_call_cb(FlMethodChannel *channel,
                           FlMethodCall *method_call,
                           gpointer user_data)
{
  const gchar *method = fl_method_call_get_name(method_call);
  if (strcmp(method, "my_method") == 0) {
      // Code for "my_method"
  } else {
    // Create response
    g_autoptr(FlMethodResponse) response = FL_METHOD_RESPONSE(fl_method_not_implemented_response_new());

    // Create error, in this case null
    g_autoptr(GError) error = nullptr;

    // Send response back to dart
    fl_method_call_respond(method_call, response, &error);

  }
}
```

### Error handling

Before we jump into arguments and custom result, let's quickly look at error handling.

Fortunately for us, it's very similar to not implemented method:

```
static void method_call_cb(FlMethodChannel *channel,
                           FlMethodCall *method_call,
                           gpointer user_data)
{
  const gchar *method = fl_method_call_get_name(method_call);
  if (strcmp(method, "my_method") == 0) {

    // An error has occurred
    g_autoptr(FlMethodResponse) response = FL_METHOD_RESPONSE(fl_method_error_response_new(
           "error_id", "Custom message", nullptr));
    
    // Create error, in this case null
    g_autoptr(GError) error = nullptr;

    // Send response back to dart
    fl_method_call_respond(method_call, response, &error);

  } else {
      // Previous not implemented method result
  }
}
```

We can see it's almost identical, we just create the result with `fl_method_error_response_new` instead of `fl_method_not_implemented_response_new`.

### Fetching Dart arguments

Alright, now it's time to write what we wanted to write from the beginning.
Let's assume that we want to send data from Dart to C++, to do that we just send a map from Dart side, but how to fetch it?

To do so, we need to call `fl_method_call_get_args(FlMethodCall)` which returns a pointer to  `FlValue`.

Next we check if returned value is the proper type:


```
static void method_call_cb(FlMethodChannel *channel,
                           FlMethodCall *method_call,
                           gpointer user_data)
{
  const gchar *method = fl_method_call_get_name(method_call);
  if (strcmp(method, "my_method") == 0)
  {
    // Get Dart arguments
    FlValue* args = fl_method_call_get_args(method_call);
    // Fetch string value named "name"
    FlValue *text_value = fl_value_lookup_string(args, "name");

    // Check if returned value is either null or string
    if (text_value == nullptr ||
        fl_value_get_type(text_value) != FL_VALUE_TYPE_STRING)
    {

      // Return error 
      g_autoptr(FlMethodResponse) response = FL_METHOD_RESPONSE(fl_method_not_implemented_response_new());
    
      // Create error, in this case null
      g_autoptr(GError) error = nullptr;

      // Send response back to dart
      fl_method_call_respond(method_call, response, &error);
    } else {
        // Custom logic
    }

  }
}
```

In example above we lookup for string, but there are other ones like int, float, bool, map. For complete list checkout [Flutter Engine Documentation](https://engine.chinmaygarde.com/fl__value_8cc.html)

Same goes for type check, to check whole list of enum go to [Documentation](https://engine.chinmaygarde.com/fl__value_8h.html#ae514203eea95fec354c1648957f77a4d)

### Returning some value

We're almost done, we managed to handle not implemented method, errors, and getting arguments from method. What's left is returning values back to Dart.

Fortunately for us it's very similar to what we've already done, we just need to create `FlMethodResponse` and put `FlValue` inside. Here is an example:


```
static void method_call_cb(FlMethodChannel *channel,
                           FlMethodCall *method_call,
                           gpointer user_data)
{
  const gchar *method = fl_method_call_get_name(method_call);
  if (strcmp(method, "my_method") == 0)
  {

    // Fetch string value named "name"
    FlValue *text_value = fl_value_lookup_string(args, "name");

    // Check if returned value is either null or string
    if (text_value == nullptr ||
        fl_value_get_type(text_value) != FL_VALUE_TYPE_STRING)
    {
        // Previous error handling
    } else {


        // Create response
        FlValue *res = fl_value_new_string("Response from Linux");

        // Send it back to Dart
        g_autoptr(FlMethodResponse) response = FL_METHOD_RESPONSE(fl_method_success_response_new(res));
    }

  }
}
```


Like before here is [link to documentation for more value creation function](https://engine.chinmaygarde.com/fl__value_8h.html#a1b2a2f35eb78370fbb2d1af774f6d245)


# VSCode code completion + debugging

I think I should give you some reward for coming this far, so I decided to write setup for code completion and debugging in Visual Studio Code.

Before you start, [C++](https://marketplace.visualstudio.com/items?itemName=ms-vscode.cpptools) and [Cmake](https://marketplace.visualstudio.com/items?itemName=ms-vscode.cmake-tools) plugins must be installed.

First let's setup code completion. To do that create file named `c_cpp_properties.json` inside `.vscode` folder in the root of your project and put this config inside:

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

Check your compiler path (Flutter uses Clang) and adjust C/C++ standards if need.

To setup debugging we need to create launch configuration inside `launch.json` in the same `.vscode` folder. Let's take a look at the config:

```
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Debug native",
            "type": "cppdbg",
            "request": "launch",
            "program": "${workspaceFolder}/build/linux/x64/debug/bundle/hello_world",   // ADJUST YOUR BINARY NAME HERE
            "cwd": "${workspaceFolder}"
        }
    ]
}
```

Pretty straightforward but you need to change your binary name.
Also, be aware that to make it work you need to build your flutter project with `flutter run`.


# Closing

Thank you for reading, hopefully you will find it useful. 

Happy Coding!

Complete example can be found [here](https://github.com/charafau/linux_method_channel).