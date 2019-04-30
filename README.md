# org.openhab.binding.kberry

All following steps I'm running under linux, javac version 1.8.0_212.

## Patching calimero-core

```text
git clone https://github.com/calimero-project/calimero-core.git
cd calimero-core
git checkout e9975c0
git checkout -b kberry
vim src/tuwien/auto/calimero/link/KNXNetworkLinkFT12.java
git commit -am "patch for kberry support"
./gradlew clean
./gradlew jar
cp build/libs/calimero-core-2.4-SNAPSHOT.jar ~/calimero-core-2.4-kberry.jar
```

Explaining:

1. git checkout commands rolls back code to commit e9975c0 as in original org.openhab.binding.knx package. So, the only difference in calimero-core lib will be following patch.

In new versions calimero uses Java 11. I decided to build binding under version 8.

2. vim used for edit file. Following modifications are made:

```text
diff --git a/src/tuwien/auto/calimero/link/KNXNetworkLinkFT12.java b/src/tuwien/auto/calimero/link/KNXNetworkLinkFT12.java
index 1984662..81d738c 100644
--- a/src/tuwien/auto/calimero/link/KNXNetworkLinkFT12.java
+++ b/src/tuwien/auto/calimero/link/KNXNetworkLinkFT12.java
@@ -96,7 +96,7 @@ public class KNXNetworkLinkFT12 extends AbstractLink<FT12Connection>
        protected KNXNetworkLinkFT12(final FT12Connection c, final KNXMediumSettings settings) throws KNXException
        {
                super(c, c.getPortID(), settings);
-               cEMI = false;
+               cEMI = true;
                sendCEmiAsByteArray = true;
                linkLayerMode();
                conn.addConnectionListener(notifier);
@@ -140,7 +140,8 @@ public class KNXNetworkLinkFT12 extends AbstractLink<FT12Connection>
        private void linkLayerMode() throws KNXException
        {
                // link layer mode
-               final byte[] switchLinkLayer = { (byte) PEI_SWITCH, 0x00, 0x18, 0x34, 0x56, 0x78, 0x0A, };
+               // final byte[] switchLinkLayer = { (byte) PEI_SWITCH, 0x00, 0x18, 0x34, 0x56, 0x78, 0x0A, };
+               final byte[] switchLinkLayer = { (byte) 0xf6, 0x00, 0x08, 0x01, 0x34, 0x10, 0x01, 0x00 };
                try {
                        conn.send(switchLinkLayer, true);
                }
@@ -157,7 +158,8 @@ public class KNXNetworkLinkFT12 extends AbstractLink<FT12Connection>

        private void normalMode() throws KNXAckTimeoutException, KNXPortClosedException, InterruptedException
        {
-               final byte[] switchNormal = { (byte) PEI_SWITCH, 0x1E, 0x12, 0x34, 0x56, 0x78, (byte) 0x9A, };
+               //final byte[] switchNormal = { (byte) PEI_SWITCH, 0x1E, 0x12, 0x34, 0x56, 0x78, (byte) 0x9A, };
+               final byte[] switchNormal = { (byte) 0xf6, 0x00, 0x08, 0x01, 0x34, 0x10, 0x01, (byte) 0xf0 };
                conn.send(switchNormal, true);
        }
 }

```

change cEMI to true, and, accordingly with [KNX BAOS User Guide](https://weinzierl.de/images/download/development/830/KnxBAOS_Users_Guide.pdf) page 45 5.2.2, send telegram to switch BAOS module to link layer mode. Also, add message to switch back to BAOS mode in normalMode() method. Without switching messages(only cEMI = true), I got warnings in openhab logs:

```text
01:36:30.297 [WARN ] [calimero.link./dev/ttyS1             ] - received unspecified frame f0c100010001000118020726
```

`f0c1` means `DatapointValue.Ind` baos service, so it was easy to understand that swithing command is required.

3. Commit, assemble jar and copy to home directory. I will add it as a dependency to binding jar

## Building openhab binding

1. Install eclipse following [instructions](https://www.openhab.org/docs/developer/development/ide.html)
2. Assuming eclipse was installed at `~/openhab-master` folder.

Start it and wait it for performing background tasks. 

```text
~/openhab-master/eclipse/eclipse
```

After it is done, there will be cloned git `openhab-master/git/openhab2-addons` repository.

3. Go to this folder and look around. There is no knx binding source code. So, switch to 2.4 and create new branch

```text
git checkout 2.4.0
git checkout -b kberry
```

now, there is `addons/binding/org.openhab.knx` folder

4. In eclipse package explorer right click on `OH2 Addons` -> Import. "General/Existing Projects into Workspace", then select root directory `/home/username/openhab-master/git/openhab2-addons/addons/binding`, In projects list deselect all and select only knx binding `org.openhab.binding.knx`.

5. Open imported project tree, go to Referenced Libraries. There is calimero-core-2.4-e9975c01.jar. Right click on it -> Build Path -> Remove from Build Path.

6. Delete this file from `lib folder`. Copy patched ~/calimero-core-2.4-kberry.jar instead. Right click on copied file -> Build Path -> Add to Build Path.

7. Replace filename in `build.properties`. Instead of `lib/calimero-core-2.4-e9975c01.jar` there should be `lib/calimero-core-2.4-kberry.jar`.

8. Do the same for `META-INF/MANIFEST.MF`.

9. Optional. Refactor binding name to "KNX Binding(kBerry)" in `ESH-INF/binding/binding.xml`

10. In Package Explorer right click on `org.openhab.binding.knx` folder -> Export. Select Java -> JAR file -> Next. Select checkbox "Export generated class files and resources", "Export Java Source files and resources", "Compress content of JAR file", "Add directories entries". Select path to save for your JAR file. Click Next. I did not change selected "export with compile errors" checkbox on next page. Proceeded next. On last page, select radio button "Use existing manifest from workspace". Select `META-INF/MANIFEST.MF`. Now click finish.

## checking

copy generated file to `/usr/share/openhab2/addons/` on machine where openhab2 is installed and to which BAOS module is connected.

Change ownership

```text
sudo chown openhab: /usr/share/openhab2/addons/org.openhab.binding.kberry.jar
```

Openhab2 should find this binding automatically. If needed, restart openhab2.

11. Created new serial port bridge thing. It gaves me communication error "The serial FT1.2 KNX connection requires the RXTX libraries to be available, but they could not be found!"

```text
sudo apt install librxtx-java
```

To be sure openhab user has access to serialport:

```text
sudo usermod -a -G dialout openhab
```

Finally, I was able to fix this issue by installing Serial binding, Modbus binding from Paper UI. It brings required dependencies with it.
