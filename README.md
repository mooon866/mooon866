package com.flash.nowifi

import android.Manifest
import android.content.pm.PackageManager
import android.os.Bundle
import android.os.Environment
import android.widget.*
import androidx.appcompat.app.AppCompatActivity
import androidx.core.app.ActivityCompat
import androidx.core.content.ContextCompat
import com.downloader.OnDownloadListener
import com.downloader.PRDownloader
import com.downloader.PRDownloaderConfig

class MainActivity : AppCompatActivity() {

    private lateinit var categorySpinner: Spinner
    private lateinit var downloadButton: Button
    private lateinit var pauseButton: Button
    private lateinit var resumeButton: Button
    private lateinit var progressBar: ProgressBar
    private lateinit var progressText: TextView
    private lateinit var downloadedCountText: TextView

    private var downloadId: Int = 0
    private var videosDownloadedCount = 0

    private val maxVideos = 200

    private val videoCategories = mapOf(
        "رياضة" to listOf(
            "https://sample-videos.com/video123/mp4/720/big_buck_bunny_720p_1mb.mp4",
            "https://sample-videos.com/video123/mp4/720/big_buck_bunny_720p_2mb.mp4"
        ),
        "ترفيه" to listOf(
            "https://sample-videos.com/video123/mp4/720/big_buck_bunny_720p_3mb.mp4"
        ),
        "تعليم" to listOf(
            "https://sample-videos.com/video123/mp4/720/big_buck_bunny_720p_4mb.mp4"
        )
    )

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        categorySpinner = findViewById(R.id.categorySpinner)
        downloadButton = findViewById(R.id.downloadButton)
        pauseButton = findViewById(R.id.pauseButton)
        resumeButton = findViewById(R.id.resumeButton)
        progressBar = findViewById(R.id.progressBar)
        progressText = findViewById(R.id.progressText)
        downloadedCountText = findViewById(R.id.downloadedCountText)

        val adapter = ArrayAdapter(this, android.R.layout.simple_spinner_item, videoCategories.keys.toList())
        adapter.setDropDownViewResource(android.R.layout.simple_spinner_dropdown_item)
        categorySpinner.adapter = adapter

        if (ContextCompat.checkSelfPermission(this, Manifest.permission.WRITE_EXTERNAL_STORAGE)
            != PackageManager.PERMISSION_GRANTED) {
            ActivityCompat.requestPermissions(this, arrayOf(Manifest.permission.WRITE_EXTERNAL_STORAGE), 1)
        }

        val config = PRDownloaderConfig.newBuilder()
            .setDatabaseEnabled(true)
            .build()
        PRDownloader.initialize(applicationContext, config)

        downloadButton.setOnClickListener {
            val selectedCategory = categorySpinner.selectedItem.toString()
            val videos = videoCategories[selectedCategory] ?: emptyList()
            if (videos.isNotEmpty()) {
                if (videosDownloadedCount >= maxVideos) {
                    Toast.makeText(this, "لقد وصلت للحد الأقصى من التحميلات ($maxVideos)", Toast.LENGTH_LONG).show()
                } else {
                    startDownloading(videos[0])
                }
            }
        }

        pauseButton.setOnClickListener {
            PRDownloader.pause(downloadId)
        }

        resumeButton.setOnClickListener {
            PRDownloader.resume(downloadId)
        }
    }

    private fun startDownloading(url: String) {
        val filePath = getExternalFilesDir(Environment.DIRECTORY_MOVIES)?.absolutePath
        val fileName = "${System.currentTimeMillis()}.mp4"

        downloadId = PRDownloader.download(url, filePath, fileName)
            .build()
            .setOnStartOrResumeListener {
                Toast.makeText(this, "بدأ التحميل...", Toast.LENGTH_SHORT).show()
            }
            .setOnPauseListener {
                Toast.makeText(this, "تم الإيقاف المؤقت.", Toast.LENGTH_SHORT).show()
            }
            .setOnCancelListener {
                Toast.makeText(this, "تم إلغاء التحميل.", Toast.LENGTH_SHORT).show()
            }
            .setOnProgressListener { progress ->
                val progressPercent = (progress.currentBytes * 100 / progress.totalBytes).toInt()
                progressBar.progress = progressPercent
                progressText.text = "تقدم التحميل: $progressPercent%"
            }
            .start(object : OnDownloadListener {
                override fun onDownloadComplete() {
                    videosDownloadedCount++
                    downloadedCountText.text = "عدد الفيديوهات المحمّلة: $videosDownloadedCount"
                    Toast.makeText(this@MainActivity, "اكتمل التحميل!", Toast.LENGTH_SHORT).show()
                }

                override fun onError(error: com.downloader.Error?) {
                    Toast.makeText(this@MainActivity, "خطأ أثناء التحميل!", Toast.LENGTH_SHORT).show()
                }
            })
    }
}
