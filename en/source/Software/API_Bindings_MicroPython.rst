
.. _api_bindings_micropython:

MicroPython - API Bindings
==========================

The MicroPython bindings allow you to control :ref:`Bricks <primer_bricks>` and
:ref:`Bricklets <primer_bricklets>` from MicroPython scripts running on
microcontroller boards such as ESP32, Raspberry Pi Pico and others. The
:ref:`ZIP file <downloads_bindings_examples>` for the bindings contains:

* in ``source/`` the source code of the bindings as flat ``.py`` files
* in ``examples/`` the examples for every Brick and Bricklet
* in ``stubs/`` the ``.pyi`` type stub files for IDE code completion

Requirements
------------

* `MicroPython <https://micropython.org/>`__ 1.17 or newer
* **TCP/IP mode**: A board with WiFi or Ethernet networking (e.g. ESP32,
  Raspberry Pi Pico W) and a Brick Daemon or WiFi/Ethernet Extension
* **Local SPI mode**: A board with Bricklets connected directly via SPI
  (e.g. ESP32 Brick, ESP32 Ethernet Brick, Raspberry Pi with HAT).
  No network connection needed.

.. _api_bindings_micropython_install:

Installation
------------

Since MicroPython boards have limited filesystems and do not support pip or
setuptools, there is no package installer. Instead, copy the binding files
directly onto your board.

Copy the ``.py`` files from the ``source/`` folder of the
:ref:`ZIP file <downloads_bindings_examples>` to your board using a tool such as
`mpremote <https://docs.micropython.org/en/latest/reference/mpremote.html>`__,
`Thonny <https://thonny.org/>`__ or
`ampy <https://github.com/scientificit/ampy>`__. For example, using mpremote::

 mpremote cp source/connection_common.py :
 mpremote cp source/ip_connection.py :
 mpremote cp source/bricklet_temperature_v2.py :

Copy only the bindings you actually need to save space on the board. At
minimum, you always need ``connection_common.py`` plus the connection module
for your mode:

* **TCP/IP mode**: ``connection_common.py`` + ``ip_connection.py`` + device
  bindings
* **Local SPI mode**: ``connection_common.py`` + ``spi_connection.py`` + a
  HAL module (e.g. ``hal_esp32_brick.py``) + device bindings

WiFi Setup
----------

For WiFi-capable boards (e.g. ESP32), the network connection must be
established before connecting to a Brick Daemon. This can be done using
MicroPython's ``network`` module:

.. code-block:: python

  import network

  wlan = network.WLAN(network.STA_IF)
  wlan.active(True)
  wlan.connect("YOUR_SSID", "YOUR_PASSWORD")

  while not wlan.isconnected():
      pass

  print("Connected:", wlan.ifconfig())

hmac Module (for Authentication)
--------------------------------

If you want to use authentication (``ipcon.authenticate()``), the ``hmac``
module is required. Most MicroPython builds do not include it by default.
Install it using MicroPython's package manager (requires a network connection)::

 import mip
 mip.install("hmac")

Testing an Example
------------------

To test a MicroPython example :ref:`Brick Daemon <brickd>` and :ref:`Brick
Viewer <brickv>` have to be installed first. Brick Daemon acts as a proxy
between the USB interface of the Bricks and the API bindings. Brick Viewer
connects to Brick Daemon and helps to figure out basic information about the
connected Bricks and Bricklets.

As an example let's test the configuration example for the Stepper Brick.
For this copy the ``example_configuration.py`` file from the
``examples/brick/stepper/`` folder and the required binding files to your
board::

 board/
  -> ip_connection.py
  -> brick_stepper.py
  -> example_configuration.py

In the example ``HOST`` and ``PORT`` specify at which network address the
Stepper Brick can be found. If it is connected locally to USB then ``localhost``
and 4223 is correct. When running on a microcontroller board, replace
``localhost`` with the IP address of the computer running Brick Daemon. The
``UID`` value has to be changed to the UID of the connected Stepper Brick,
which you can figure out using Brick Viewer:

.. code-block:: python

  HOST = "192.168.1.100"
  PORT = 4223
  UID = "XXYYZZ" # Change XXYYZZ to the UID of your Stepper Brick

Now you're ready to run this example on your board::

 mpremote run example_configuration.py

.. note::
 Unlike the regular Python bindings, MicroPython bindings use a flat module
 structure. Imports use the form ``from ip_connection import IPConnection``
 instead of ``from tinkerforge.ip_connection import IPConnection``.

.. note::
 The MicroPython bindings use a synchronous/polling architecture. There is no
 background thread for callback dispatch. You must call
 ``ipcon.dispatch_callbacks(seconds)`` periodically to handle incoming
 callbacks. Use a negative value for infinite dispatching:
 ``ipcon.dispatch_callbacks(-1)``.

.. note::
 Auto-reconnect is not supported in the MicroPython bindings because it
 requires background threads. You must handle reconnection explicitly in your
 code.

.. _api_bindings_micropython_spi:

Local SPI Connection
--------------------

For boards that have Bricklets connected directly via SPI — such as the
ESP32 Brick, ESP32 Ethernet Brick, or a Raspberry Pi with HAT — you can use
``SPIConnection`` instead of ``IPConnection`` to access Bricklets without any
network connection. This uses the SPITFP (SPI Tinkerforge Protocol) to
communicate directly over the SPI bus.

**Advantages over TCP/IP mode:**

* No network stack required (no WiFi, no socket, no Brick Daemon)
* Lower latency (direct SPI access)
* Smaller footprint (no ``ip_connection.py``, ``hashlib`` or ``hmac`` needed)

**Minimum files needed on the board:**

* ``connection_common.py`` — shared device and protocol primitives
* ``spi_connection.py`` — SPI connection with SPITFP protocol
* A HAL module for your board (e.g. ``hal_esp32_brick.py``)
* The device binding(s) you want to use

**Available HAL modules:**

* ``hal_esp32_brick.py`` — ESP32 Brick (6 ports A-F, dual SPI bus)
* ``hal_esp32_ethernet_brick.py`` — ESP32 Ethernet Brick (6 ports, demux CS)
* ``hal_raspberry_pi.py`` — Raspberry Pi (user-configurable CS pins)
* ``hal_linux.py`` — Linux boards with spidev
* ``hal_generic.py`` — Any MicroPython board (fully user-configurable)

**Example** (ESP32 Brick with a Temperature Bricklet 2.0):

.. code-block:: python

  from hal_esp32_brick import ESP32BrickHAL
  from spi_connection import SPIConnection
  from bricklet_temperature_v2 import BrickletTemperatureV2

  spi = SPIConnection(ESP32BrickHAL())
  spi.connect()

  t = BrickletTemperatureV2('ABC', spi)  # use UID of your Bricklet
  print('Temperature:', t.get_temperature() / 100.0, 'C')

  spi.disconnect()

``SPIConnection`` provides the same API as ``IPConnection`` — the same
``send_request()``, ``dispatch_callbacks()``, ``enumerate()`` and
``register_callback()`` methods. Device bindings work unchanged with either
connection type.

**Enumerating Bricklets** on the SPI bus:

.. code-block:: python

  from hal_esp32_brick import ESP32BrickHAL
  from spi_connection import SPIConnection

  def cb_enumerate(uid, connected_uid, position, hw_version, fw_version,
                   device_id, enumeration_type):
      print('UID: {}, Port: {}, Device ID: {}'.format(uid, position, device_id))

  spi = SPIConnection(ESP32BrickHAL())
  spi.connect()
  spi.register_callback(SPIConnection.CALLBACK_ENUMERATE, cb_enumerate)
  spi.enumerate()
  spi.disconnect()

**Using the Generic HAL** for custom boards:

.. code-block:: python

  from hal_generic import GenericHAL
  from spi_connection import SPIConnection

  hal = GenericHAL(ports=[
      {'name': 'A', 'cs_pin': 16, 'spi_id': 2},
      {'name': 'B', 'cs_pin': 17, 'spi_id': 2},
  ])
  spi = SPIConnection(hal)
  spi.connect()

Reducing File Size with mpy-cross
----------------------------------

The ``.py`` binding files can be compiled to MicroPython bytecode (``.mpy``
files) to reduce file size and speed up import times. This is done using the
``mpy-cross`` tool.

.. important::
 The ``mpy-cross`` version must match the MicroPython firmware version on your
 board. For example, if your board runs MicroPython 1.23.x, you need
 ``mpy-cross`` from the 1.23 release. Using a mismatched version will result
 in import errors on the board.

Install ``mpy-cross`` matching your firmware version::

 pip install mpy-cross==1.23.0  # adjust version to match your firmware

Compile a binding file::

 mpy-cross bricklet_temperature_v2.py

This creates ``bricklet_temperature_v2.mpy`` in the same directory. Copy the
``.mpy`` file to your board instead of the ``.py`` file. All imports work the
same way — MicroPython automatically finds ``.mpy`` files.

To compile all binding files at once::

 for f in source/*.py; do mpy-cross "$f"; done

Typical size reduction is around 70-80% compared to the original ``.py`` files.

IDE Support with Type Stubs
---------------------------

The ZIP file includes a ``stubs/`` folder with ``.pyi`` type stub files for all
bindings. These provide code completion, type checking and inline documentation
in IDEs such as VS Code (with Pylance) or PyCharm.

To use the stubs, configure your IDE to include the ``stubs/`` folder as an
extra analysis path. For VS Code, add the following to your ``.vscode/settings.json``:

.. code-block:: json

 {
   "python.analysis.extraPaths": ["path/to/stubs"]
 }

The stubs contain full type annotations and docstrings for all device classes,
methods and constants. They are not needed on the board itself — they are only
used by the IDE during development.

API Reference and Examples
--------------------------

Links to the API reference for the IP Connection, Bricks and Bricklets as
well as the examples from the ZIP file of the bindings are listed in the
following table. Further project descriptions can be found in the
:ref:`Kits <index_kits>` section.

.. include:: API_Bindings_MicroPython_links.table

.. toctree::
   :hidden:

   IP Connection <IPConnection_MicroPython>
   Bricks <Bricks_MicroPython>
   Bricks (Discontinued) <Bricks_MicroPython_Discontinued>
   Bricklets <Bricklets_MicroPython>
   Bricklets (Discontinued) <Bricklets_MicroPython_Discontinued>
