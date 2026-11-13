class UsbCopyService : Service() {

    companion object {
        const val PREFS = "usb_copy_prefs"
        const val DEST_KEY = "usb_destination"
        private const val CHANNEL_ID = "usb_copy_channel"
        private const val NOTIFICATION_ID = 4107
        private const val POLL_MS = 8000L

        private val SOURCES = linkedMapOf(
            "DCIM" to "src_DCIM",
            "Pictures" to "src_Pictures",
            "Movies" to "src_Movies",
            "Download" to "src_Download",
            "Documents" to "src_Documents",
            "Music" to "src_Music",
            "Recordings" to "src_Recordings"
        )
    }

    private val running = AtomicBoolean(false)
    @Volatile private var worker: Thread? = null
    private val prefs by lazy { getSharedPreferences(PREFS, Context.MODE_PRIVATE) }

    override fun onCreate() {
        super.onCreate()
        createNotificationChannel()
        startForeground(NOTIFICATION_ID, buildNotification("Memantau flashdisk USB…"))
    }

    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        if (running.compareAndSet(false, true)) {
            worker = Thread { monitorUsb() }.also {
                it.name = "UsbAutoCopyMonitor"
                it.start()
            }
        }
        return START_STICKY
    }

    override fun onBind(intent: Intent?): IBinder? = null

    override fun onDestroy() {
        running.set(false)
        worker?.interrupt()
        super.onDestroy()
    }

    private fun monitorUsb() {
        var copiedForCurrentMount = false
        var lastUsbPresent = false

        while (running.get()) {
            try {
                val usbPresent = hasMountedRemovableVolume()
                if (!usbPresent) {
                    copiedForCurrentMount = false
                    updateNotification("Menunggu flashdisk USB…")
                } else if (!lastUsbPresent) {
                    copiedForCurrentMount = false
                    updateNotification("Flashdisk terdeteksi. Memeriksa izin…")
                }

                if (usbPresent && !copiedForCurrentMount) {
                    val destinationUri = prefs.getString(DEST_KEY, null)
                    val destination = destinationUri?.let {
                        DocumentFile.fromTreeUri(this, Uri.parse(it))
                    }

                    if (destination != null && destination.exists() && destination.canWrite()) {
                        copiedForCurrentMount = true
                        copyAllSources(destination)
                    } else {
                        updateNotification(
                            "USB terdeteksi, tetapi folder tujuan belum tersedia. Pilih ulang folder USB di aplikasi."
                        )
                    }
                }

                lastUsbPresent = usbPresent
                Thread.sleep(POLL_MS)
            } catch (_: InterruptedException) {
                break
            } catch (e: Exception) {
                updateNotification("Menunggu USB / izin: ${e.message ?: "periksa flashdisk"}")
                try { Thread.sleep(POLL_MS) } catch (_: InterruptedException) { break }
            }
        }
    }

    private fun hasMountedRemovableVolume(): Boolean {
        val storageManager = getSystemService(Context.STORAGE_SERVICE) as StorageManager
        return storageManager.storageVolumes.any { volume ->
            volume.isRemovable && volume.state == Environment.MEDIA_MOUNTED
        }
    }

    private fun copyAllSources(destination: DocumentFile) {
        val powerManager = getSystemService(Context.POWER_SERVICE) as PowerManager
        val wakeLock = powerManager.newWakeLock(
            PowerManager.PARTIAL_WAKE_LOCK,
            "USBAutoCopy::CopyWakeLock"
        )

        var copied = 0
        var skipped = 0
        var failed = 0

        try {
            wakeLock.acquire(30 * 60 * 1000L)

            SOURCES.forEach { (folderName, prefKey) ->
                val sourceUri = prefs.getString(prefKey, null)
                if (sourceUri == null) {
                    failed++
                    return@forEach
                }

                val source = DocumentFile.fromTreeUri(this, Uri.parse(sourceUri))
                if (source == null || !source.exists() || !source.canRead()) {
                    failed++
                    return@forEach
                }

                val targetFolder = destination.findFile(folderName)
                    ?.takeIf { it.isDirectory }
                    ?: destination.createDirectory(folderName)

                if (targetFolder == null) {
                    failed++
                } else {
                    val result = copyDirectoryContents(source, targetFolder, folderName)
                    copied += result.copied
                    skipped += result.skipped
                    failed += result.failed
                }
                updateNotification("Menyalin… Berhasil $copied, dilewati $skipped, gagal $failed")
            }

            updateNotification("Selesai. Berhasil $copied, dilewati $skipped, gagal $failed")
        } catch (e: Exception) {
            updateNotification("Penyalinan terhenti: ${e.message ?: "kesalahan"}")
        } finally {
            if (wakeLock.isHeld) wakeLock.release()
        }
    }

    private data class CopyResult(
        var copied: Int = 0,
        var skipped: Int = 0,
        var failed: Int = 0
    ) {
        fun add(other: CopyResult) {
            copied += other.copied
            skipped += other.skipped
            failed += other.failed
        }
    }

    private fun copyDirectoryContents(
        source: DocumentFile,
        destination: DocumentFile,
        relativePath: String
    ): CopyResult {
        val result = CopyResult()

        for (item in source.listFiles()) {
            try {
                val name = item.name
                if (name == null) {
                    result.failed++
                    continue
                }

                if (item.isDirectory) {
                    val targetDir = destination.findFile(name)
                        ?.takeIf { it.isDirectory }
                        ?: destination.createDirectory(name)

                    if (targetDir == null) {
                        result.failed++
                    } else {
                        result.add(copyDirectoryContents(item, targetDir, "$relativePath/$name"))
                    }
                } else if (item.isFile) {
                    val existing = destination.findFile(name)
                    if (existing != null) {
                        result.skipped++
                        continue
                    }

                    val mime = item.type ?: "application/octet-stream"
                    val tempName = ".$name.usbcopy-${UUID.randomUUID()}.tmp"
                    val tempFile = destination.createFile("application/octet-stream", tempName)

                    if (tempFile == null) {
                        result.failed++
                        continue
                    }

                    try {
                        contentResolver.openInputStream(item.uri).use { inStream ->
                            if (inStream == null) throw IOException("Tidak dapat membaca $name")
                            contentResolver.openOutputStream(tempFile.uri, "w").use { outStream ->
                                if (outStream == null) throw IOException("Tidak dapat menulis $name")
                                inStream.copyTo(outStream, 64 * 1024)
                                outStream.flush()
                            }
                        }

                        // Rename only after a complete write, so an interrupted
                        // transfer is not mistaken for a completed destination file.
                        if (destination.findFile(name) == null && tempFile.renameTo(name)) {
                            result.copied++
                        } else {
                            tempFile.delete()
                            result.failed++
                        }
                    } catch (e: Exception) {
                        tempFile.delete()
                        result.failed++
                    }
                }
            } catch (_: Exception) {
                result.failed++
            }
        }

        return result
    }

    private fun createNotificationChannel() {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            val channel = NotificationChannel(
                CHANNEL_ID,
                "USB Auto Copy",
                NotificationManager.IMPORTANCE_LOW
            ).apply {
                description = "Status pemantauan dan penyalinan flashdisk"
            }
            getSystemService(NotificationManager::class.java)
                .createNotificationChannel(channel)
        }
    }

    private fun buildNotification(message: String): Notification {
        val openApp = PendingIntent.getActivity(
            this,
            0,
            Intent(this, MainActivity::class.java),
            PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE
        )

        return NotificationCompat.Builder(this, CHANNEL_ID)
            .setSmallIcon(android.R.drawable.stat_sys_upload)
            .setContentTitle("USB Auto Copy")
            .setContentText(message)
            .setStyle(NotificationCompat.BigTextStyle().bigText(message))
            .setContentIntent(openApp)
            .setOngoing(true)
            .setOnlyAlertOnce(true)
            .build()
    }

    private fun updateNotification(message: String) {
        val manager = getSystemService(NotificationManager::class.java)
        manager.notify(NOTIFICATION_ID, buildNotification(message))
    }
}
