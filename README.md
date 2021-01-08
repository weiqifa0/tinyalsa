weiqifa@bsp-ubuntu1804:~/is13-sdk$ git diff external/tinyalsa/
diff --git a/external/tinyalsa/Android.bp b/external/tinyalsa/Android.bp
old mode 100644
new mode 100755
index 090d91c0f8..79a6ceaee2
--- a/external/tinyalsa/Android.bp
+++ b/external/tinyalsa/Android.bp
@@ -9,6 +9,7 @@ cc_library {
         "mixer.c",
         "pcm.c",
     ],
+    shared_libs: ["liblog"],
     cflags: ["-Werror", "-Wno-macro-redefined"],
     export_include_dirs: ["include"],
     local_include_dirs: ["include"],
diff --git a/external/tinyalsa/pcm.c b/external/tinyalsa/pcm.c
old mode 100644
new mode 100755
index 4ae321bf93..de0deab9d0
--- a/external/tinyalsa/pcm.c
+++ b/external/tinyalsa/pcm.c
@@ -35,11 +35,17 @@
 #include <unistd.h>
 #include <poll.h>

+#include <stdint.h>
+#include <signal.h>
+#include <time.h>
+
 #include <sys/ioctl.h>
 #include <sys/mman.h>
 #include <sys/time.h>
 #include <limits.h>

+#include <android/log.h>
+
 #include <linux/ioctl.h>
 #define __force
 #define __bitwise
@@ -48,6 +54,10 @@

 #include <tinyalsa/asoundlib.h>

+#define LOG_TAG "TINYALSA_QIFA"
+#define ALOGD(fmt, args...) __android_log_print(ANDROID_LOG_DEBUG, LOG_TAG, fmt, ##args)
+#define ALOGE(fmt, args...) __android_log_print(ANDROID_LOG_ERROR, LOG_TAG, fmt, ##args)
+
 #define PARAM_MAX SNDRV_PCM_HW_PARAM_LAST_INTERVAL

 /* Logs information into a string; follows snprintf() in that
@@ -242,6 +252,10 @@ static void param_init(struct snd_pcm_hw_params *p)

 struct pcm {
     int fd;
+    //int fd_test;
+    FILE *fd_test;
+    unsigned int size_test;
+    void *test_buffer;
     unsigned int flags;
     int running:1;
     int prepared:1;
@@ -566,6 +580,7 @@ int pcm_read(struct pcm *pcm, void *data, unsigned int count)
     x.frames = count / (pcm->config.channels *
                         pcm_format_to_bits(pcm->config.format) / 8);

+    ALOGE("x.frames:%ld count:%d",x.frames,count);
     for (;;) {
         if (!pcm->running) {
             if (pcm_start(pcm) < 0) {
@@ -583,6 +598,12 @@ int pcm_read(struct pcm *pcm, void *data, unsigned int count)
             }
             return oops(pcm, errno, "cannot read stream data");
         }
+
+        if (fwrite(x.buf, 1, count, pcm->fd_test) != count) {
+            fprintf(stderr,"Error capturing sample\n");
+            ALOGE("Error capturing sample");
+        }
+
         return 0;
     }
 }
@@ -864,6 +885,8 @@ int pcm_close(struct pcm *pcm)

     if (pcm->fd >= 0)
         close(pcm->fd);
+    if (pcm->fd_test >= 0)
+        fclose(pcm->fd_test);
     pcm->prepared = 0;
     pcm->running = 0;
     pcm->buffer_size = 0;
@@ -872,6 +895,8 @@ int pcm_close(struct pcm *pcm)
     return 0;
 }

+
+
 struct pcm *pcm_open(unsigned int card, unsigned int device,
                      unsigned int flags, struct pcm_config *config)
 {
@@ -900,7 +925,12 @@ struct pcm *pcm_open(unsigned int card, unsigned int device,
         oops(pcm, errno, "cannot open device '%s'", fn);
         return pcm;
     }
-
+    /*weiqifa*/
+    pcm->fd_test = fopen("/sdcard/ref_temp.pcm", "wb");
+    if (!pcm->fd_test) {
+        fprintf(stderr, "Unable to create file /sdcard/ref_temp.pcm\n");
+    } else ALOGD("creat /sdcard/ref_temp.pcm file success");
+
     if (fcntl(pcm->fd, F_SETFL, fcntl(pcm->fd, F_GETFL) &
               ~O_NONBLOCK) < 0) {
         oops(pcm, errno, "failed to reset blocking mode '%s'", fn);
(END)
