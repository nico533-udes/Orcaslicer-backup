# Why `bambu_networking` violates the AGPL in Bambu Studio

This text describes the problem with `bambu_networking` based on the public Bambu Studio source code.

This is not about whether Bambu Lab must allow every fork into its cloud. That is a separate topic. This is about something simpler: Bambu Studio is an AGPL v3 program, and its public code shows that the closed `bambu_networking` component is downloaded, installed, dynamically loaded and used as an integral part of the program's operation.

In my opinion, this is an AGPL compliance problem, because the AGPL requires the "Corresponding Source" of the program to be provided, and the definition of "Corresponding Source" also includes the source code of shared libraries and dynamically linked subprograms if the program is specifically designed to require them through intimate data communication or control flow.

That is exactly the kind of situation visible in the Bambu Studio code.

---

## 1. Bambu Studio is AGPL, and `bambu_networking` is described as non-free

In the Bambu Studio README, Bambu Lab itself writes that:

- Bambu Studio is licensed under GNU AGPL v3
- Bambu Studio is based on PrusaSlicer, which is AGPL v3
- PrusaSlicer is based on Slic3r, which is AGPL v3
- the `bambu networking plugin` is based on non-free libraries
- without the plugin, SD card printing remains available after slicing

Source:

https://github.com/bambulab/BambuStudio/blob/e8c7dc1b84f5e3816238e070e04eeeb67cd92783/README.md#L42-L52

This is the starting point of the whole problem.

Bambu Studio is not a closed-source program written from scratch. It is an AGPL program derived from AGPL code. Bambu Lab cannot simply move a key part of the runtime into a closed binary and call it a "plugin" if that plugin is technically designed as part of the AGPL program's operation.

---

## 2. `bambu_networking` does not look like an independent add-on

The word "plugin" itself does not decide anything. What matters is the real architecture of the program.

The public code shows that `bambu_networking` is not an external tool independently launched by the user. Bambu Studio:

- knows the exact library name
- knows the network agent version
- downloads the plugin from Bambu servers
- installs the plugin into the Bambu Studio data directory
- loads `bambu_networking.dll`, `libbambu_networking.so` or `libbambu_networking.dylib`
- resolves `bambu_network_*` function symbols
- passes C++ structures and callbacks to the plugin
- allows the plugin to execute work on the main UI thread
- uses the plugin for login, monitoring, LAN/cloud print, MakerWorld/MakerLab, camera, MQTT/device messaging, presets, filaments, telemetry and file transfer
- versions the plugin against the Bambu Studio version
- updates the plugin through an OTA mechanism

This is not accidental compatibility. This is a designed runtime component.

---

## 3. The library name and agent version are written into the AGPL code

The public Bambu Studio header contains:

```cpp
#define BAMBU_NETWORK_LIBRARY               "bambu_networking"
#define BAMBU_NETWORK_AGENT_NAME            "bambu_network_agent"
#define BAMBU_NETWORK_AGENT_VERSION         "02.06.01.50"
```

Source:

https://github.com/bambulab/BambuStudio/blob/e8c7dc1b84f5e3816238e070e04eeeb67cd92783/src/slic3r/Utils/bambu_networking.hpp#L103-L106

This shows that the AGPL program does not have a generic mechanism for "any plugin". It knows a specific Bambu component, a specific name and a specific version.

This matters for the AGPL because it shows that the program is specifically designed for this component.

---

## 4. The AGPL code defines a shared ABI for the closed plugin

The public `bambu_networking.hpp` header defines callback types passed between Bambu Studio and the plugin:

```cpp
typedef std::function<void(int online_login, bool login)> OnUserLoginFn;
typedef std::function<void(std::string topic_str)> OnPrinterConnectedFn;
typedef std::function<void(int status, std::string dev_id, std::string msg)> OnLocalConnectedFn;
typedef std::function<void(int return_code, int reason_code)> OnServerConnectedFn;
typedef std::function<void(std::string dev_id, std::string msg)> OnMessageFn;
typedef std::function<void(unsigned http_code, std::string http_body)> OnHttpErrorFn;
typedef std::function<void(int status, int code, std::string msg)> OnUpdateStatusFn;
typedef std::function<bool()> WasCancelledFn;
typedef std::function<bool(int status, std::string job_info)> OnWaitFn;
typedef std::function<void(std::string dev_info_json_str)> OnMsgArrivedFn;
typedef std::function<void(std::function<void()>)> QueueOnMainFn;
```

Source:

https://github.com/bambulab/BambuStudio/blob/e8c7dc1b84f5e3816238e070e04eeeb67cd92783/src/slic3r/Utils/bambu_networking.hpp#L123-L145

The most important one here is `QueueOnMainFn`.

This is not a simple text parameter. It is a callback that allows the plugin to pass a function to be executed on Bambu Studio's main UI thread. Such a mechanism means control flow from the plugin back into the AGPL application.

This means that `bambu_networking` is not an independent application. It is woven into the program's control flow.

---

## 5. The AGPL code defines data structures used by the plugin

`bambu_networking.hpp` defines, among other things, `PrintParams`. This structure contains fields related to the printer, project, files, FTP, MD5, nozzle mapping, AMS mapping, connection type, printer IP, SSL, login, password, calibration, timelapse and AMS flags.

Source:

https://github.com/bambulab/BambuStudio/blob/e8c7dc1b84f5e3816238e070e04eeeb67cd92783/src/slic3r/Utils/bambu_networking.hpp#L197-L247

There are also other structures shared with the plugin:

- `detectResult`: https://github.com/bambulab/BambuStudio/blob/e8c7dc1b84f5e3816238e070e04eeeb67cd92783/src/slic3r/Utils/bambu_networking.hpp#L186-L195
- `TaskQueryParams`: https://github.com/bambulab/BambuStudio/blob/e8c7dc1b84f5e3816238e070e04eeeb67cd92783/src/slic3r/Utils/bambu_networking.hpp#L249-L255
- `FilamentQueryParams`: https://github.com/bambulab/BambuStudio/blob/e8c7dc1b84f5e3816238e070e04eeeb67cd92783/src/slic3r/Utils/bambu_networking.hpp#L257-L265
- `FilamentDeleteParams`: https://github.com/bambulab/BambuStudio/blob/e8c7dc1b84f5e3816238e070e04eeeb67cd92783/src/slic3r/Utils/bambu_networking.hpp#L267-L271
- `PublishParams`: https://github.com/bambulab/BambuStudio/blob/e8c7dc1b84f5e3816238e070e04eeeb67cd92783/src/slic3r/Utils/bambu_networking.hpp#L273-L280
- `MessageFlag`: https://github.com/bambulab/BambuStudio/blob/e8c7dc1b84f5e3816238e070e04eeeb67cd92783/src/slic3r/Utils/bambu_networking.hpp#L291-L296

This is exactly the type of situation described by the AGPL: intimate data communication between the program and a dynamically linked component.

---

## 6. Bambu Studio dynamically loads `bambu_networking`

In `NetworkAgent::initialize_network_module()`, Bambu Studio builds the path to the `plugins` directory and loads the library:

- Windows: `bambu_networking.dll`
- macOS: `libbambu_networking.dylib`
- Linux: `libbambu_networking.so`

Source:

https://github.com/bambulab/BambuStudio/blob/e8c7dc1b84f5e3816238e070e04eeeb67cd92783/src/slic3r/Utils/NetworkAgent.cpp#L184-L281

This is dynamic linking designed in the AGPL application's code.

It is not an external application launched as a separate process through a standard system interface. It is a library loaded into the Bambu Studio process.

---

## 7. Bambu Studio resolves 108 functions from the closed library

`NetworkAgent.cpp` calls `get_network_function("bambu_network_...")` 108 times.

These include functions for, among other things:

- creating and destroying the agent
- starting the agent
- logging in the user
- checking login status
- getting user id and user name
- connecting to the server
- subscribing to MQTT/device messages
- sending messages
- connecting to the printer
- printing through the cloud
- printing through LAN
- sending G-code to the SD card
- getting the camera URL
- getting printer firmware
- handling MakerWorld/MakerLab
- managing user presets
- managing cloud filament data
- telemetry
- HMS snapshot
- publish workflow

Source:

https://github.com/bambulab/BambuStudio/blob/e8c7dc1b84f5e3816238e070e04eeeb67cd92783/src/slic3r/Utils/NetworkAgent.cpp#L284-L392

108 functions on the ABI side is not a small optional helper. It is a large application subsystem moved behind a closed binary boundary.

---

## 8. The same module provides an additional file-transfer ABI

After loading `bambu_networking`, Bambu Studio calls:

```cpp
InitFTModule(networking_module);
```

Source:

https://github.com/bambulab/BambuStudio/blob/e8c7dc1b84f5e3816238e070e04eeeb67cd92783/src/slic3r/Utils/NetworkAgent.cpp#L280-L281

`FileTransferUtils.hpp` defines an additional ABI for file transfer, including tunnel/job handles, job results, JSON payloads and binary buffers.

Sources:

- structures and handles: https://github.com/bambulab/BambuStudio/blob/e8c7dc1b84f5e3816238e070e04eeeb67cd92783/src/slic3r/Utils/FileTransferUtils.hpp#L25-L45
- tunnel/job function types: https://github.com/bambulab/BambuStudio/blob/e8c7dc1b84f5e3816238e070e04eeeb67cd92783/src/slic3r/Utils/FileTransferUtils.hpp#L71-L90
- module initialization: https://github.com/bambulab/BambuStudio/blob/e8c7dc1b84f5e3816238e070e04eeeb67cd92783/src/slic3r/Utils/FileTransferUtils.hpp#L239-L252

This additionally shows that `bambu_networking` is not only about "cloud login". It also handles parts of file transfer and the print workflow.

---

## 9. Bambu Studio downloads the plugin itself

`GUI_App::download_plugin()`:

- builds the plugin URL
- downloads metadata from the Bambu server
- extracts the plugin package URL from JSON
- downloads the plugin package
- writes it to a temporary file
- reports the result through `networkplugin_download`

Source:

https://github.com/bambulab/BambuStudio/blob/e8c7dc1b84f5e3816238e070e04eeeb67cd92783/src/slic3r/GUI/GUI_App.cpp#L1543-L1759

This matters because the plugin is not a random component that the user finds by themselves. The AGPL program contains a built-in mechanism for downloading the closed component.

---

## 10. Bambu Studio installs the plugin itself

`GUI_App::install_plugin()`:

- creates the `plugins` directory
- creates the `plugins/backup` directory
- opens the downloaded ZIP
- extracts files into the plugin directory
- copies files into backup
- sets `installed_networking` to `1`

Source:

https://github.com/bambulab/BambuStudio/blob/e8c7dc1b84f5e3816238e070e04eeeb67cd92783/src/slic3r/GUI/GUI_App.cpp#L1761-L1910

This is another proof that `bambu_networking` is managed by Bambu Studio as part of its runtime.

---

## 11. After installation, Bambu Studio restarts and wires the whole network subsystem

`GUI_App::restart_networking()`:

- calls `on_init_network(true)`
- resets `StaticBambuLib` state
- initializes callbacks
- starts discovery
- refreshes the UI
- reloads user presets after login
- starts preset synchronization

Source:

https://github.com/bambulab/BambuStudio/blob/e8c7dc1b84f5e3816238e070e04eeeb67cd92783/src/slic3r/GUI/GUI_App.cpp#L1912-L1956

This is not the behavior of an independent program. This is the behavior of a runtime component that, after installation, is immediately woven into the application state.

---

## 12. The plugin is versioned against Bambu Studio

`check_networking_version()` compares the plugin version with the Studio version.

Source:

https://github.com/bambulab/BambuStudio/blob/e8c7dc1b84f5e3816238e070e04eeeb67cd92783/src/slic3r/GUI/GUI_App.cpp#L1980-L1997

`get_plugin_url()` builds the plugin URL based on `SLIC3R_VERSION`.

Source:

https://github.com/bambulab/BambuStudio/blob/e8c7dc1b84f5e3816238e070e04eeeb67cd92783/src/slic3r/GUI/GUI_App.cpp#L1543-L1555

This shows that the plugin is matched to a specific Bambu Studio version. It is not an independent standard program, but a component version-coupled to the AGPL application.

---

## 13. At application startup, the plugin is wrapped into central Bambu Studio managers

`GUI_App::on_init_network()`:

- loads the module through `NetworkAgent::initialize_network_module()`
- checks version compatibility
- creates `NetworkAgent`
- passes `m_agent` to `DeviceManager`
- passes `m_agent` to `UserManager`
- creates `TaskManager(m_agent)`
- sets the config dir
- sets the cert dir
- initializes callbacks
- sets the country code
- starts the agent
- loads the last printer

Source:

https://github.com/bambulab/BambuStudio/blob/e8c7dc1b84f5e3816238e070e04eeeb67cd92783/src/slic3r/GUI/GUI_App.cpp#L3518-L3615

This is very strong architectural evidence. The closed plugin is not beside the application. It sits underneath central application managers.

---

## 14. Login and MakerLab/MakerWorld are blocked without a compatible plugin

For `homepage_login_or_register`, Bambu Studio checks `is_compatibility_version()`. If the plugin is not compatible, the application sends `homepage_need_networkplugin` and interrupts the login flow.

Source:

https://github.com/bambulab/BambuStudio/blob/e8c7dc1b84f5e3816238e070e04eeeb67cd92783/src/slic3r/GUI/GUI_App.cpp#L4618-L4640

A similar mechanism exists for `homepage_makerlab_open`.

Source:

https://github.com/bambulab/BambuStudio/blob/e8c7dc1b84f5e3816238e070e04eeeb67cd92783/src/slic3r/GUI/GUI_App.cpp#L4812-L4850

This shows that user account functions and MakerLab/MakerWorld are dependent on the plugin.

---

## 15. The Monitor tab is blocked without the network agent

For Bambu presets, switching to the Monitor tab is blocked if `wxGetApp().getAgent()` does not exist. The application then shows a prompt to install the network plugin.

Source:

https://github.com/bambulab/BambuStudio/blob/e8c7dc1b84f5e3816238e070e04eeeb67cd92783/src/slic3r/GUI/MainFrame.cpp#L1278-L1305

This again shows that the plugin is not only an add-on for the cloud. It conditions the main printer-monitoring function in the UI.

---

## 16. LAN print and cloud print go through `m_agent`

In `PrintJob.cpp`, the network printing paths go through `m_agent`:

- `start_sdcard_print(...)`
- `start_local_print_with_record(...)`
- `start_print(...)`
- `start_local_print(...)`

Source:

https://github.com/bambulab/BambuStudio/blob/e8c7dc1b84f5e3816238e070e04eeeb67cd92783/src/slic3r/GUI/Jobs/PrintJob.cpp#L550-L619

This is an important distinction.

I am not claiming that without the plugin Bambu Studio cannot generate a file at all or save it to an SD card. Bambu's own README indicates that the SD card workflow remains.

I am saying something different: in the public Bambu Studio code, network print submission, including the LAN workflow, is routed through `NetworkAgent`, meaning through the `bambu_networking` layer.

This means that `bambu_networking` is not only a "cloud plugin". It is a component handling the main network workflows of Bambu Studio.

---

## 17. The OTA updater treats the plugin as an official Bambu Studio resource

`PresetUpdater::get_cached_plugins_version()` expects the following files in the cache:

- `bambu_networking.dll` / `libbambu_networking.dylib` / `libbambu_networking.so`
- `BambuSource.dll` / `libBambuSource.dylib` / `libBambuSource.so`
- `live555.dll` / `liblive555.dylib` / `liblive555.so`
- `network_plugins.json`

Source:

https://github.com/bambulab/BambuStudio/blob/e8c7dc1b84f5e3816238e070e04eeeb67cd92783/src/slic3r/Utils/PresetUpdater.cpp#L1131-L1163

`PresetUpdater::sync_plugins()` downloads the `slicer/plugins/cloud` resource and, on a forced update, sets `update_network_plugin=true`.

Source:

https://github.com/bambulab/BambuStudio/blob/e8c7dc1b84f5e3816238e070e04eeeb67cd92783/src/slic3r/Utils/PresetUpdater.cpp#L1165-L1249

This means that the plugin is part of the official application update mechanism. Bambu Studio does not merely tolerate this component. It manages it.

---

## 18. Default publisher validation further ties the plugin to the application

The code contains a module publisher/certificate checking mechanism. `NetworkAgent::initialize_network_module()` can compare the certificate of the application and the module through `IsSamePublisher()`.

Sources:

- module loading and checking: https://github.com/bambulab/BambuStudio/blob/e8c7dc1b84f5e3816238e070e04eeeb67cd92783/src/slic3r/Utils/NetworkAgent.cpp#L195-L265
- default `ignore_module_cert=false`: https://github.com/bambulab/BambuStudio/blob/e8c7dc1b84f5e3816238e070e04eeeb67cd92783/src/libslic3r/AppConfig.cpp#L508-L510
- call with `!ignore_module_cert`: https://github.com/bambulab/BambuStudio/blob/e8c7dc1b84f5e3816238e070e04eeeb67cd92783/src/slic3r/GUI/GUI_App.cpp#L3520-L3541

I do not base the whole argument on this point, because the code contains an `ignore_module_cert` path. But the fact of default publisher validation itself shows that the plugin is not designed as a neutral, open interface for arbitrary implementations. It is designed as a Bambu component.

---

## 19. Why the AGPL matters here

AGPL v3 section 1 defines "Corresponding Source".

The most important part: "Corresponding Source" includes, among other things, interface definition files and the source code of shared libraries and dynamically linked subprograms that the work is specifically designed to require, for example through intimate data communication or control flow.

Official AGPL text:

https://www.gnu.org/licenses/agpl-3.0.en.html#section1

In this case, the public code shows all elements of this definition:

- the AGPL program has an interface definition for the plugin
- the AGPL program knows the specific library name
- the AGPL program dynamically loads the library
- the AGPL program resolves 108 functions from that library
- the AGPL program passes C++ structures to that library
- the AGPL program passes C++ callbacks to that library
- the plugin can return to the main UI thread through `QueueOnMainFn`
- the AGPL program uses the plugin in central application workflows
- the AGPL program downloads, installs, updates and versions the plugin itself

For this reason, `bambu_networking` fits the description of a dynamically linked component for which Corresponding Source should be available.

---

## 20. "Optional plugin" does not solve the problem

Bambu may say that the plugin is optional.

Technically, this is partly true: minimal slicer use and the SD card workflow may exist without the plugin.

But this does not settle the AGPL issue.

If an AGPL program is specifically designed to download, install, load and call a closed library through a strict ABI, callbacks, shared structures and control flow, then calling that library "optional" does not automatically make it an independent aggregate.

Optional packaging does not erase the fact that the code contains a designed runtime integration.

---

## 21. This does not look like a "mere aggregate"

The AGPL, like the GPL, distinguishes linking a program with another component from a mere aggregate on the same medium.

`bambu_networking` does not look like a mere aggregate, because:

- it is loaded into the Bambu Studio process
- it has an ABI defined in the Bambu Studio code
- it shares data structures with Bambu Studio
- it uses C++ callbacks
- it receives a callback to the main UI thread
- it has 108 function symbols used by Bambu Studio
- it is versioned with Bambu Studio
- it is installed by Bambu Studio
- it is updated by Bambu Studio
- it handles central Bambu Studio functions

The FSF GPL FAQ describes a similar situation with plugins: if plugins are dynamically linked, make function calls to each other and share data structures, then according to the FSF they form one combined program, not an independent aggregate.

Source:

https://www.gnu.org/licenses/gpl-faq.html#GPLPlugins

---

## 22. `bambu_networking` does not look like a System Library

The AGPL excludes so-called System Libraries from "Corresponding Source". But `bambu_networking` does not fit that exception.

It is not an operating system library, a standard compiler library, a system runtime or a common platform component.

The facts from the code show the opposite:

- the library is specifically named `bambu_networking`
- it is installed into the Bambu Studio plugin directory
- it is downloaded from Bambu servers
- it is versioned for Bambu Studio
- it is loaded by `NetworkAgent`
- it handles Bambu-specific functions: printers, cloud, LAN print, MakerWorld, filaments, presets, camera, telemetry

For this reason, the System Library exception is not a convincing defense for this plugin.

---

## 23. Bambu Connect does not fix the `bambu_networking` problem

Bambu Connect is a separate topic.

Even if Bambu now wants to direct external slicers to Bambu Connect instead of `bambu_networking`, that does not solve the problem in Bambu Studio.

The AGPL problem concerns the fact that Bambu Studio, as an AGPL program, is publicly designed to load and use the closed `bambu_networking`.

Changing the workflow for OrcaSlicer or other slicers does not change the fact that Bambu Studio still contains an AGPL + non-free dynamic runtime component architecture.

Bambu Connect may be a separate application, separate proxy or separate authorization tool. But it is not an answer to the question of why `bambu_networking` in Bambu Studio should be AGPL-compliant without providing its Corresponding Source.

---

## 24. This is not an argument about forcing cloud access

This point must be separated very clearly.

I am not claiming that the AGPL automatically forces Bambu to allow every modified client into their private cloud.

I am claiming something different: if Bambu distributes an AGPL program that is designed to load and use a closed runtime component through a strict ABI, then that component may fall within the program's "Corresponding Source".

This is a distribution compliance problem under the AGPL, not a simple dispute over cloud terms of service.

---

## 25. Context of restrictions introduced after printers were purchased

A separate issue is that Bambu introduced new authorization restrictions for network functions. Bambu itself publicly described that authorization controls cover, among other things:

- starting a print job through LAN/cloud
- remote video
- firmware upgrade
- motion controls
- temperature controls
- fan controls
- AMS controls
- calibration controls

At the same time, Bambu writes that starting a print from an SD card remains outside this mechanism.

Source:

https://blog.bambulab.com/firmware-update-introducing-new-authorization-control-system-2/

This is important context for users, because it shows that network functions and integrations were restricted after the printer purchase. But it is not the main proof of the AGPL violation.

The main proof of the AGPL violation is in the public Bambu Studio code and in the way `bambu_networking` is integrated.

---

## 26. USA - why this is a copyright law problem

In the United States, a copyright owner has exclusive rights including:

- reproduction of the work
- preparation of derivative works
- distribution of copies

Source:

https://www.law.cornell.edu/uscode/text/17/106

The AGPL is a copyright license. It grants permission to copy, modify and distribute under the conditions of that license.

If a distributor of an AGPL program does not comply with the license conditions, it may lose the scope of the permission granted. Then the problem is not just an "open source dispute". It may become a problem of distributing code outside the license terms.

In this case, the question is: can Bambu distribute an AGPL program designed to load the closed `bambu_networking` without providing the Corresponding Source of that component?

Based on the public code, my answer is: this should not be AGPL-compliant, unless Bambu has a separate licensing exception or permissions from all required copyright holders of the AGPL code.

---

## 27. EU - why this is a copyright law problem

In the European Union, computer programs are protected by copyright. Directive 2009/24/EC covers, among other things:

- reproduction of a program
- adaptation, translation, alteration and modification of a program
- public distribution of a program

Source:

https://www.wipo.int/wipolex/en/legislation/details/8612

Just like in the USA, the AGPL is the basis of permission to use, modify and distribute the AGPL program.

If the AGPL conditions are not met, distribution of the program may exceed the license granted.

Here, the problem is the combination of an AGPL program with a closed dynamic component, which the public Bambu Studio code shows as required for central network workflows.

---

## 28. Why Bambu cannot simply add a closed exception

Bambu Studio is based on PrusaSlicer and Slic3r, and therefore on AGPL code from many authors.

Bambu Lab may own its own changes, but that does not mean it can unilaterally change the license terms for the entire derivative program.

To legally add a closed exception for a component like `bambu_networking`, Bambu would need the appropriate permissions/licenses from all required copyright holders of the code that is part of the AGPL program.

Without such an exception, the default AGPL rule remains: the whole combined program and its Corresponding Source must be available under AGPL-compatible terms.

---

## 29. The simplest chain of evidence

1. Bambu Studio is AGPL v3.
2. Bambu's README admits that the `bambu networking plugin` is based on non-free libraries.
3. The AGPL code knows the library name `bambu_networking`.
4. The AGPL code knows the `bambu_network_agent` version.
5. The AGPL code defines a shared C++ ABI for the plugin.
6. The AGPL code defines data structures passed to the plugin.
7. The AGPL code defines callbacks passed to the plugin.
8. The plugin gets the ability to queue work onto the main UI thread.
9. Bambu Studio dynamically loads a `.dll`, `.so` or `.dylib`.
10. Bambu Studio resolves 108 `bambu_network_*` symbols.
11. The same module initializes the file-transfer ABI.
12. Bambu Studio downloads the plugin itself.
13. Bambu Studio installs the plugin itself.
14. Bambu Studio updates the plugin itself.
15. Bambu Studio versions the plugin against `SLIC3R_VERSION`.
16. Bambu Studio uses the plugin in login, monitoring, LAN/cloud print, MakerWorld/MakerLab, camera, device messaging, filaments, presets, telemetry and file transfer.
17. AGPL v3 says Corresponding Source includes dynamically linked components that the program is specifically designed to require through intimate data communication or control flow.
18. `bambu_networking` has these characteristics.
19. If Bambu distributes this component only as a closed binary without Corresponding Source under AGPL-compatible terms, this is a serious AGPL compliance problem.

---

## 30. My conclusion

Based on the public Bambu Studio code, I believe that `bambu_networking` is not an independent, loose add-on. It is a dynamically loaded runtime component specifically designed for Bambu Studio, used through a large ABI, shared data structures, callbacks, a main UI-thread callback, file-transfer ABI, an installation/update mechanism and central application functions.

AGPL v3 requires Corresponding Source for such components if the program is specifically designed to require them through intimate data communication or control flow.

Therefore, in my opinion, distributing Bambu Studio as AGPL together with the closed `bambu_networking` without providing its complete Corresponding Source under AGPL-compatible terms violates the AGPL.

No reverse engineering of the plugin is needed for this. The public Bambu Studio code is enough to show how deeply `bambu_networking` is integrated with the AGPL program.

