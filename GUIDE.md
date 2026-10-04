# Self-hosted Rose party mode by 0x\!/rep4ir

## ☣️DISCLAIMER: Even if this is really easy, this is NOT intended to be a beginner friendly guide as I will assume that you have at least a basic knowledge of the topics I am going to talk about in this file. So if you don't know what an IDE or Python is, this guide might not be for you, if anything my DM’s are closed and will continue to be, so if you need help ask ChatGPT or something.

# Prerequisites:

- [Python 3.11 or newer](https://www.python.org/downloads/release/python-3110/)  
- [Anaconda / Miniconda](https://www.anaconda.com/download)  
- [Visual Studio Build Tools with the .NET desktop build tools, WPF support, and the .NET Framework 4.7.2 targeting pack (for Pengu Loader), and the MSVC C++ build tools (for Rose's stand-in cslol-dll.dll)](https://visualstudio.microsoft.com/downloads/)  
- [Inno Setup 6 (if you also want to create the installer](https://jrsoftware.org/isdl.php))  
- [Domain (you can get one for free from deSEC)](https://desec.io/)  
- [Git](https://git-scm.com/)

# Step-by-Step Guide

## 1\. Create and activate a Conda environment

Rose requires Python 3.11. Create and activate the environment:

| PS C:\\Rose\> conda create \-n rose python=3.11 \-y |
| :---- |

| PS C:\\Rose\> conda activate rose |
| :---- |

## 2\. Clone the Rose repository

| PS C:\\Rose\> git clone https\://github.com/Alban1911/Rose.git |
| :---- |

| PS C:\\Rose\\rose\> cd Rose |
| :---- |

## 3\. Install Python dependencies

| PS C:\\Rose\\rose\> pip install \-r requirements.txt |
| :---- |

## 4\. Deploy the relay worker to Cloudflare

Navigate to the worker directory and deploy it. The "relay-worker/package.json" contains a "deploy" script that wraps "wrangler deploy".

| PS C:\\Rose\\rose\> cd relay-worker |
| :---- |

| PS C:\\Rose\\rose\> npm install \# Install worker dependencies (first time only) |
| :---- |

| PS C:\\Rose\\rose\> npm run deploy |
| :---- |

Wrangler will authenticate you with Cloudflare (opening a browser window on first run) and deploy the worker. On success, it prints the deployed URL, which looks like:

| https\://rose-party-relay.\<your-subdomain\>.workers.dev |
| :---- |

**Copy this URL** — you need it in the next step.

## 5\. Configure the relay URL in Rose

The client resolves the relay URL from two sources, in order:

1. The "ROSE\_RELAY\_URL" environment variable.  
2. A local "relay\_config.py" file inside "party/network/" containing a "RELAY\_URL" constant.

The "relay\_config.py" file is git-ignored, so it is the intended place for a private URL. Create it:

| PS C:\\Rose\\rose\> cd ../party/network |
| :---- |

Create the file "relay\_config.py" with the following content (replace the URL with your actual worker URL):

| RELAY\_URL \= "wss://rose-party-relay.\<your-subdomain\>.workers.dev" |
| :---- |

**Important:** Use the "wss://" protocol, not "https\://". The relay is a WebSocket server. The client appends "/room?key=\<room\_key\>" to this base URL when connecting.

## 6\. Build the Pengu Loader plugins (optional, for development)

Rose builds the Pengu Loader executable from the vendored source in vendor/PenguLoader-1.1.6/ during packaging. A prebuilt Pengu Loader.exe is intentionally not committed to the repository, you can build only that component:

| PS C:\\Rose\\rose\> python scripts/build\_pengu\_loader.py |
| :---- |

**Note:** The build scripts needed in the scripts\\ repository are, "build\_all.py", "build\_pyinstaller.py", "create\_installer.py" and "build\_pengu\_loader.py".

## 7\. Build the Rose executable

To build the standalone executable (which also rebuilds the Pengu Loader and the "cslol-dll.dll" stand-in):

| PS C:\\Rose\\rose\\party\\network\> cd ../.. |
| :---- |

| PS C:\\Rose\\rose\> python scripts/build\_pyinstaller.py |
| :---- |

This produces "dist/Rose/Rose.exe" and its supporting folder.

## 8\. Build Rose and the Windows installer

To build everything and create the Windows installer ("installer/Rose\_Setup\_\<version\>.exe"):

| PS C:\\Rose\\rose\> python scripts/build\_all.py |
| :---- |

This script runs "build\_pyinstaller.py" and then "create\_installer.py" (which uses Inno Setup) to produce the final installer.

## 9\. Create the auto-updater package

To zip the built "dist/Rose" folder into the auto-updater's package format ("installer/update\_package\_\<version\>.zip"):

| PS C:\\Rose\\rose\> python scripts/create\_update\_package.py |
| :---- |

# Verification

After deploying the worker, you can verify it is running with a simple health check:

| PS C:\\Rose\\rose\> curl https://rose-party-relay.\<your-subdomain\>.workers.dev |
| :---- |

The worker's "fetch" handler responds to a "GET" request at the root path with a JSON status:

| {"status":"ok","service":"rose-party-relay"} |
| :---- |

If you see this JSON, the worker is deployed correctly. To test the WebSocket path end-to-end, launch Rose (with the "relay\_config.py" in place), enable Party Mode, and share the generated room key with another Rose user.

# How Party Mode Works (Context)

\- The relay uses a Cloudflare **Durable Object** ("PartyRoom") to maintain state for each room. The "wrangler.toml" binds the "ROOM" namespace to this class.  
\- When a client connects, it opens a WebSocket to "wss://\<your-worker-url\>/room?key=\<room\_key\>".  
\- The relay broadcasts the full member list on every change (join, leave, skin update) and enforces a **maximum of 10 members per room**.  
\- Clients older than Rose **1.4.4** are rejected with an HTTP "426" status.