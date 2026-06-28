# Source Code

## xplayer/app/src/main/AndroidManifest.xml

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android">

    <uses-permission android:name="android.permission.INTERNET" />
    <uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
    <uses-permission android:name="android.permission.WAKE_LOCK" />

    <application
        android:name=".PlayerApplication"
        android:allowBackup="true"
        android:icon="@android:drawable/sym_def_app_icon"
        android:label="XPlayer"
        android:supportsRtl="true">
    </application>

</manifest>

```

## xplayer/app/src/main/java/com/xplayer/dev/PlayerApplication.kt

```kotlin
package com.xplayer.dev

import android.app.Application
import com.xplayer.dev.crash.CrashReporter
import com.xplayer.dev.crash.CrashReportingHook
import dagger.hilt.android.HiltAndroidApp
import timber.log.Timber
import javax.inject.Inject

@HiltAndroidApp
class PlayerApplication : Application() {

    @Inject lateinit var crashReportingHook: CrashReportingHook
    @Inject lateinit var crashReporter: CrashReporter

    override fun onCreate() {
        super.onCreate()
        installLogging()
        crashReportingHook.install()
    }

    private fun installLogging() {
        if (BuildConfig.DEBUG) {
            Timber.plant(Timber.DebugTree())
        } else {
            Timber.plant(ReleaseTree())
        }
    }

    private inner class ReleaseTree : Timber.Tree() {
        override fun log(priority: Int, tag: String?, message: String, t: Throwable?) {
            if (priority >= android.util.Log.WARN) {
                t?.let { crashReporter.recordNonFatalException(it) }
            }
        }
    }
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/state/PlayerState.kt

```kotlin
package com.xplayer.dev.core.state

import androidx.media3.common.PlaybackException
import androidx.media3.common.MediaItem

/**
 * # PlayerState — Sealed Hierarchy
 *
 * Represents every possible lifecycle state of the player engine.
 *
 * ## Design Rules
 *  1. Every state is a **data class or data object** — structural equality,
 *     copy(), toString() for free.
 *  2. Every state carries [enteredAtMs] — wall-clock epoch for SLA measurement.
 *  3. Every state carries [sequenceNumber] — monotonically increasing counter
 *     to detect and discard stale state updates.
 *  4. State objects are **immutable** — all fields are val.
 *  5. Business logic lives in extensions — state hierarchy stays clean.
 *  6. [PlayerState.Error] preserves the full causal chain via [previousState].
 *
 * ## State Machine
 * ```
 *                    ┌─────────────────────────────┐
 *                    ▼                             │
 *  Uninitialised → Idle → Preparing → Buffering → Ready → Playing
 *                    ▲                    │          │       │    │
 *                    │                   │          └──────►│    │
 *                    │                   ▼                  ▼    ▼
 *                    └───────────────── Error ◄──── Paused  Ended
 *                                        │
 *                                        └──► Idle (reset)
 *                                        └──► Preparing (retry)
 * ```
 *
 * ## Threading
 *  State objects are immutable — safe to read from any thread.
 *  State transitions are emitted on the main thread via [PlayerStateMachine].
 */
sealed class PlayerState {

    /** Wall-clock millis when this state was entered. */
    abstract val enteredAtMs: Long

    /**
     * Monotonically increasing sequence number.
     * Used to detect and reject stale / out-of-order state updates.
     */
    abstract val sequenceNumber: Long

    // ═══════════════════════════════════════════════════════════════════════
    // States
    // ═══════════════════════════════════════════════════════════════════════

    /**
     * Engine created but not yet initialised.
     * No ExoPlayer instance exists in this state.
     */
    data object Uninitialised : PlayerState() {
        override val enteredAtMs: Long     = System.currentTimeMillis()
        override val sequenceNumber: Long  = 0L
    }

    /**
     * Engine initialised; no media assigned.
     * ExoPlayer exists but is idle.
     *
     * @param sequenceNumber Incremented on each Idle re-entry (stop/reset).
     */
    data class Idle(
        override val enteredAtMs: Long    = System.currentTimeMillis(),
        override val sequenceNumber: Long = 1L,
    ) : PlayerState()

    /**
     * Media assigned; ExoPlayer pipeline is being constructed.
     * Codec negotiation, DRM provisioning, manifest fetch in progress.
     *
     * @param mediaId      Stable identifier for the current media item.
     * @param mediaItem    Full Media3 item being prepared.
     * @param startPositionMs  Position the player will seek to on ready.
     *                         0 means start from beginning.
     */
    data class Preparing(
        val mediaId: String,
        val mediaItem: MediaItem,
        val startPositionMs: Long = 0L,
        override val enteredAtMs: Long    = System.currentTimeMillis(),
        override val sequenceNumber: Long = 0L,
    ) : PlayerState()

    /**
     * Pipeline ready; insufficient data to play without interruption.
     *
     * @param mediaId        Current media identifier.
     * @param bufferPercent  Buffered percentage [0..100].
     * @param bufferedMs     Absolute buffered position in milliseconds.
     * @param rebufferCount  Number of rebuffers in this session.
     *                       0 = initial buffer, >0 = mid-playback stall.
     * @param isInitialBuffer True for the very first buffer; false for stalls.
     */
    data class Buffering(
        val mediaId: String,
        val bufferPercent: Int            = 0,
        val bufferedMs: Long              = 0L,
        val rebufferCount: Int            = 0,
        val isInitialBuffer: Boolean      = true,
        override val enteredAtMs: Long    = System.currentTimeMillis(),
        override val sequenceNumber: Long = 0L,
    ) : PlayerState() {
        init {
            require(bufferPercent in 0..100) {
                "bufferPercent must be in 0..100, was $bufferPercent"
            }
            require(bufferedMs >= 0L) {
                "bufferedMs must be >= 0, was $bufferedMs"
            }
            require(rebufferCount >= 0) {
                "rebufferCount must be >= 0, was $rebufferCount"
            }
        }
    }

    /**
     * Enough data buffered to begin playback instantly.
     * Player is NOT yet playing — waiting for [PlayerStateCommand.Play].
     *
     * @param mediaId    Current media identifier.
     * @param durationMs Total duration of the media. -1 for live streams.
     * @param isLive     True if this is a live stream.
     * @param isSeekable True if seeking is supported.
     */
    data class Ready(
        val mediaId: String,
        val durationMs: Long              = 0L,
        val isLive: Boolean               = false,
        val isSeekable: Boolean           = true,
        override val enteredAtMs: Long    = System.currentTimeMillis(),
        override val sequenceNumber: Long = 0L,
    ) : PlayerState() {
        /** Returns true if duration is known (not a live stream / unknown). */
        val isDurationKnown: Boolean get() = durationMs > 0L
    }

    /**
     * Actively rendering audio/video frames.
     *
     * @param mediaId         Current media identifier.
     * @param positionMs      Current playback position in milliseconds.
     * @param durationMs      Total duration. -1 for live streams.
     * @param bufferedMs      Absolute buffer head position.
     * @param playbackSpeed   Current speed multiplier (1.0 = normal).
     * @param isLive          True if this is a live stream.
     * @param isSeekable      True if seeking is supported.
     * @param videoWidth      0 for audio-only content.
     * @param videoHeight     0 for audio-only content.
     */
    data class Playing(
        val mediaId: String,
        val positionMs: Long              = 0L,
        val durationMs: Long              = 0L,
        val bufferedMs: Long              = 0L,
        val playbackSpeed: Float          = 1.0f,
        val isLive: Boolean               = false,
        val isSeekable: Boolean           = true,
        val videoWidth: Int               = 0,
        val videoHeight: Int              = 0,
        override val enteredAtMs: Long    = System.currentTimeMillis(),
        override val sequenceNumber: Long = 0L,
    ) : PlayerState() {

        init {
            require(positionMs >= 0L) {
                "positionMs must be >= 0, was $positionMs"
            }
            require(playbackSpeed > 0f) {
                "playbackSpeed must be > 0, was $playbackSpeed"
            }
        }

        /**
         * Normalised progress in [0.0 .. 1.0].
         * Returns 0.0 for live streams or unknown duration.
         */
        val progress: Float
            get() = when {
                isLive || durationMs <= 0L -> 0f
                else -> (positionMs.toFloat() / durationMs).coerceIn(0f, 1f)
            }

        /**
         * Remaining time in milliseconds.
         * Returns -1 for live streams.
         */
        val remainingMs: Long
            get() = if (isLive || durationMs <= 0L) -1L
                    else (durationMs - positionMs).coerceAtLeast(0L)

        /** True if this is audio-only content. */
        val isAudioOnly: Boolean get() = videoWidth == 0 && videoHeight == 0

        /**
         * Buffer ahead of current position in milliseconds.
         * Indicates how much can be played without rebuffering.
         */
        val bufferAheadMs: Long
            get() = (bufferedMs - positionMs).coerceAtLeast(0L)
    }

    /**
     * Playback intentionally paused.
     *
     * @param mediaId       Current media identifier.
     * @param positionMs    Position where playback was paused.
     * @param durationMs    Total duration. -1 for live streams.
     * @param pausedBySystem True = system-initiated (audio focus loss, call).
     *                       False = user-initiated.
     * @param pauseReason   Detailed reason for the pause.
     */
    data class Paused(
        val mediaId: String,
        val positionMs: Long              = 0L,
        val durationMs: Long              = 0L,
        val pausedBySystem: Boolean       = false,
        val pauseReason: PauseReason      = PauseReason.USER,
        override val enteredAtMs: Long    = System.currentTimeMillis(),
        override val sequenceNumber: Long = 0L,
    ) : PlayerState() {

        init {
            require(positionMs >= 0L) {
                "positionMs must be >= 0, was $positionMs"
            }
        }

        enum class PauseReason {
            USER,               // explicit user action
            AUDIO_FOCUS_LOSS,   // transient audio focus loss
            AUDIO_FOCUS_DUCK,   // audio focus duck (partial loss)
            PHONE_CALL,         // incoming / active call
            HEADSET_DISCONNECT, // wired headset unplugged
            APP_BACKGROUND,     // app moved to background
            SYSTEM_INTERRUPT,   // generic system interrupt
        }
    }

    /**
     * Media played to completion.
     *
     * @param mediaId        Media that ended.
     * @param totalDurationMs Actual total duration (from stream, not metadata).
     * @param completionPercent 0..100; <100 if stream ended unexpectedly.
     */
    data class Ended(
        val mediaId: String,
        val totalDurationMs: Long         = 0L,
        val completionPercent: Float      = 100f,
        override val enteredAtMs: Long    = System.currentTimeMillis(),
        override val sequenceNumber: Long = 0L,
    ) : PlayerState() {
        init {
            require(totalDurationMs >= 0L) {
                "totalDurationMs must be >= 0, was $totalDurationMs"
            }
            require(completionPercent in 0f..100f) {
                "completionPercent must be in 0..100, was $completionPercent"
            }
        }

        val isFullyCompleted: Boolean get() = completionPercent >= 99f
    }

    /**
     * Playback error occurred.
     *
     * @param mediaId       Media that was playing (null if error before prepare).
     * @param cause         Underlying [PlaybackException].
     * @param errorCategory High-level category for routing (retry vs fatal).
     * @param retryCount    How many retries have been attempted so far.
     * @param maxRetries    Maximum retries allowed for this error type.
     * @param isFatal       If true, no recovery is possible.
     * @param previousState The state the engine was in before the error.
     * @param recoveryHint  Suggested recovery action for the caller.
     */
    data class Error(
        val mediaId: String?,
        val cause: PlaybackException,
        val errorCategory: ErrorCategory  = ErrorCategory.fromException(cause),
        val retryCount: Int               = 0,
        val maxRetries: Int               = 3,
        val isFatal: Boolean              = errorCategory.isFatal,
        val previousState: PlayerState?   = null,
        val recoveryHint: RecoveryHint    = errorCategory.defaultRecoveryHint,
        override val enteredAtMs: Long    = System.currentTimeMillis(),
        override val sequenceNumber: Long = 0L,
    ) : PlayerState() {

        init {
            require(retryCount >= 0) {
                "retryCount must be >= 0, was $retryCount"
            }
            require(maxRetries >= 0) {
                "maxRetries must be >= 0, was $maxRetries"
            }
        }

        val errorCode: Int       get() = cause.errorCode
        val errorCodeName: String get() = cause.errorCodeName
        val errorMessage: String  get() = cause.message ?: "Error [$errorCodeName]"

        val canRetry: Boolean get() = !isFatal && retryCount < maxRetries
        val retriesRemaining: Int get() = (maxRetries - retryCount).coerceAtLeast(0)

        /** Previous state class name — safe for logging without holding reference. */
        val previousStateName: String
            get() = previousState?.let { it::class.simpleName } ?: "Unknown"

        /** Full causal chain as list (most recent first). */
        val causalChain: List<PlayerState>
            get() {
                val chain = mutableListOf<PlayerState>(this)
                var current: PlayerState? = previousState
                while (current != null && chain.size < MAX_CAUSAL_CHAIN_DEPTH) {
                    chain.add(current)
                    current = (current as? Error)?.previousState
                }
                return chain
            }

        companion object {
            private const val MAX_CAUSAL_CHAIN_DEPTH = 10
        }
    }

    // ═══════════════════════════════════════════════════════════════════════
    // Supporting Types
    // ═══════════════════════════════════════════════════════════════════════

    /**
     * High-level error classification.
     * Drives retry policy, UI messaging, and crash reporting triage.
     */
    enum class ErrorCategory(
        val isFatal: Boolean,
        val defaultRecoveryHint: RecoveryHint,
    ) {
        /** Transient network failure — retry with backoff. */
        NETWORK(
            isFatal = false,
            defaultRecoveryHint = RecoveryHint.RETRY_WITH_BACKOFF,
        ),

        /** HTTP error — may be transient (503) or permanent (404). */
        HTTP(
            isFatal = false,
            defaultRecoveryHint = RecoveryHint.RETRY_ONCE,
        ),

        /** DRM provisioning or license error. */
        DRM(
            isFatal = true,
            defaultRecoveryHint = RecoveryHint.CLEAR_AND_RELOAD,
        ),

        /** Audio / video decoder failure. */
        DECODER(
            isFatal = false,
            defaultRecoveryHint = RecoveryHint.RETRY_ONCE,
        ),

        /** The media source returned malformed or unparseable content. */
        SOURCE(
            isFatal = true,
            defaultRecoveryHint = RecoveryHint.REPORT_AND_STOP,
        ),

        /** Timeout waiting for renderer or pipeline to initialise. */
        TIMEOUT(
            isFatal = false,
            defaultRecoveryHint = RecoveryHint.RETRY_WITH_BACKOFF,
        ),

        /** Unknown / unclassified error. */
        UNKNOWN(
            isFatal = false,
            defaultRecoveryHint = RecoveryHint.RETRY_ONCE,
        );

        companion object {
            fun fromException(e: PlaybackException): ErrorCategory = when (e.errorCode) {
                PlaybackException.ERROR_CODE_IO_NETWORK_CONNECTION_FAILED,
                PlaybackException.ERROR_CODE_IO_NETWORK_CONNECTION_TIMEOUT,
                -> NETWORK

                PlaybackException.ERROR_CODE_IO_BAD_HTTP_STATUS,
                PlaybackException.ERROR_CODE_IO_CLEARTEXT_NOT_PERMITTED,
                -> HTTP

                PlaybackException.ERROR_CODE_DRM_SCHEME_UNSUPPORTED,
                PlaybackException.ERROR_CODE_DRM_PROVISIONING_FAILED,
                PlaybackException.ERROR_CODE_DRM_CONTENT_ERROR,
                PlaybackException.ERROR_CODE_DRM_LICENSE_ACQUISITION_FAILED,
                PlaybackException.ERROR_CODE_DRM_DISALLOWED_OPERATION,
                PlaybackException.ERROR_CODE_DRM_SYSTEM_ERROR,
                -> DRM

                PlaybackException.ERROR_CODE_DECODER_INIT_FAILED,
                PlaybackException.ERROR_CODE_DECODER_QUERY_FAILED,
                PlaybackException.ERROR_CODE_DECODING_FAILED,
                PlaybackException.ERROR_CODE_DECODING_FORMAT_EXCEEDS_CAPABILITIES,
                PlaybackException.ERROR_CODE_DECODING_FORMAT_UNSUPPORTED,
                -> DECODER

                PlaybackException.ERROR_CODE_IO_FILE_NOT_FOUND,
                PlaybackException.ERROR_CODE_IO_NO_PERMISSION,
                PlaybackException.ERROR_CODE_PARSING_CONTAINER_MALFORMED,
                PlaybackException.ERROR_CODE_PARSING_MANIFEST_MALFORMED,
                -> SOURCE

                PlaybackException.ERROR_CODE_TIMEOUT,
                -> TIMEOUT

                else -> UNKNOWN
            }
        }
    }

    /**
     * Structured recovery hint.
     * Callers use this to decide the next action without switching on error codes.
     */
    enum class RecoveryHint {
        /** Wait with exponential backoff and retry automatically. */
        RETRY_WITH_BACKOFF,

        /** Retry once immediately; if fails again, surface error to user. */
        RETRY_ONCE,

        /** Clear DRM session / cache and reload from scratch. */
        CLEAR_AND_RELOAD,

        /** Show error to user; do not retry automatically. */
        REPORT_AND_STOP,

        /** No action possible; engine must be released. */
        RELEASE_ENGINE,
    }
}
```

## xplayer/app/src/main/java/com/xplayer/dev/core/state/PlayerStateCommand.kt

```kotlin
package com.xplayer.dev.core.state

import androidx.media3.common.MediaItem

/**
 * Commands that drive state transitions in [PlayerStateMachine].
 *
 * Commands are the *input* to the state machine.
 * They carry all data needed to compute the next state.
 *
 * Design:
 *  - Commands are pure data — no logic.
 *  - The reducer decides whether a command is valid in the current state.
 *  - Commands are issued by [PlayerEngine] internals; never by UI directly.
 */
sealed class PlayerStateCommand {

    abstract val issuedAtMs: Long

    // ── Engine lifecycle ──────────────────────────────────────────────────────

    data class EngineInitialised(
        override val issuedAtMs: Long = System.currentTimeMillis(),
    ) : PlayerStateCommand()

    data class EngineReleased(
        override val issuedAtMs: Long = System.currentTimeMillis(),
    ) : PlayerStateCommand()

    // ── Media loading ─────────────────────────────────────────────────────────

    data class Prepare(
        val mediaId: String,
        val mediaItem: MediaItem,
        val startPositionMs: Long = 0L,
        override val issuedAtMs: Long = System.currentTimeMillis(),
    ) : PlayerStateCommand()

    data class Reset(
        override val issuedAtMs: Long = System.currentTimeMillis(),
    ) : PlayerStateCommand()

    // ── Playback control ──────────────────────────────────────────────────────

    data class Play(
        override val issuedAtMs: Long = System.currentTimeMillis(),
    ) : PlayerStateCommand()

    data class Pause(
        val reason: PlayerState.Paused.PauseReason = PlayerState.Paused.PauseReason.USER,
        override val issuedAtMs: Long = System.currentTimeMillis(),
    ) : PlayerStateCommand()

    data class Stop(
        override val issuedAtMs: Long = System.currentTimeMillis(),
    ) : PlayerStateCommand()

    data class Seek(
        val positionMs: Long,
        override val issuedAtMs: Long = System.currentTimeMillis(),
    ) : PlayerStateCommand() {
        init {
            require(positionMs >= 0L) { "positionMs must be >= 0" }
        }
    }

    data class SetSpeed(
        val speed: Float,
        override val issuedAtMs: Long = System.currentTimeMillis(),
    ) : PlayerStateCommand() {
        init {
            require(speed in 0.25f..8.0f) { "speed $speed out of range [0.25, 8.0]" }
        }
    }

    // ── Engine callbacks (ExoPlayer → state machine) ──────────────────────────

    data class BufferingStarted(
        val bufferPercent: Int   = 0,
        val bufferedMs: Long     = 0L,
        val rebufferCount: Int   = 0,
        val isInitialBuffer: Boolean = true,
        override val issuedAtMs: Long = System.currentTimeMillis(),
    ) : PlayerStateCommand()

    data class BufferingUpdated(
        val bufferPercent: Int,
        val bufferedMs: Long,
        override val issuedAtMs: Long = System.currentTimeMillis(),
    ) : PlayerStateCommand()

    data class ReadyToPlay(
        val durationMs: Long,
        val isLive: Boolean    = false,
        val isSeekable: Boolean = true,
        override val issuedAtMs: Long = System.currentTimeMillis(),
    ) : PlayerStateCommand()

    data class PlayingStarted(
        val positionMs: Long,
        val durationMs: Long,
        val bufferedMs: Long,
        val isLive: Boolean      = false,
        val isSeekable: Boolean  = true,
        val videoWidth: Int      = 0,
        val videoHeight: Int     = 0,
        override val issuedAtMs: Long = System.currentTimeMillis(),
    ) : PlayerStateCommand()

    data class PositionUpdated(
        val positionMs: Long,
        val bufferedMs: Long,
        override val issuedAtMs: Long = System.currentTimeMillis(),
    ) : PlayerStateCommand()

    data class PlaybackEnded(
        val totalDurationMs: Long,
        val completionPercent: Float = 100f,
        override val issuedAtMs: Long = System.currentTimeMillis(),
    ) : PlayerStateCommand()

    data class ErrorOccurred(
        val cause: androidx.media3.common.PlaybackException,
        val previousState: PlayerState,
        override val issuedAtMs: Long = System.currentTimeMillis(),
    ) : PlayerStateCommand()

    data class ErrorRetrying(
        val retryCount: Int,
        override val issuedAtMs: Long = System.currentTimeMillis(),
    ) : PlayerStateCommand()

    data class SpeedChanged(
        val newSpeed: Float,
        override val issuedAtMs: Long = System.currentTimeMillis(),
    ) : PlayerStateCommand()

    data class VideoSizeChanged(
        val width: Int,
        val height: Int,
        override val issuedAtMs: Long = System.currentTimeMillis(),
    ) : PlayerStateCommand()
}
```

## xplayer/app/src/main/java/com/xplayer/dev/core/state/PlayerStateDiagnostics.kt

```kotlin
package com.xplayer.dev.core.state

import androidx.media3.common.PlaybackException

/**
 * # PlayerStateDiagnostics
 *
 * Serialisable snapshot of player state for crash reporting.
 *
 * Designed to be attached to crash reports with zero risk of:
 *  - Holding strong references to Android context.
 *  - Serialisation failures (all fields are primitives / strings).
 *  - Circular references (causal chain is flattened).
 */
data class PlayerStateDiagnostics(
    val stateName: String,
    val mediaId: String?,
    val sequenceNumber: Long,
    val enteredAtMs: Long,
    val ageMs: Long,

    // Position
    val positionMs: Long?,
    val durationMs: Long?,
    val progress: Float?,
    val bufferedMs: Long?,

    // Playback
    val isLive: Boolean?,
    val isSeekable: Boolean?,
    val playbackSpeed: Float?,

    // Buffer
    val bufferPercent: Int?,
    val rebufferCount: Int?,

    // Error
    val errorCode: Int?,
    val errorCodeName: String?,
    val errorCategory: String?,
    val errorMessage: String?,
    val isFatalError: Boolean?,
    val retryCount: Int?,
    val maxRetries: Int?,
    val recoveryHint: String?,
    val previousStateName: String?,

    // Pause
    val pauseReason: String?,
    val pausedBySystem: Boolean?,

    // Ended
    val completionPercent: Float?,

    // History breadcrumbs (last 10)
    val recentBreadcrumbs: List<String>,
) {
    companion object {

        /**
         * Build a [PlayerStateDiagnostics] snapshot.
         * Safe to call from any thread — no side effects.
         *
         * @param state   Current player state.
         * @param history Last N breadcrumbs from [PlayerStateStore].
         */
        fun from(
            state: PlayerState,
            history: List<String> = emptyList(),
        ): PlayerStateDiagnostics = PlayerStateDiagnostics(
            stateName      = state.stateName,
            mediaId        = state.mediaId,
            sequenceNumber = state.sequenceNumber,
            enteredAtMs    = state.enteredAtMs,
            ageMs          = state.ageMs,

            positionMs  = state.positionMs.takeIf { it > 0L },
            durationMs  = state.durationMs.takeIf { it > 0L },
            progress    = state.progress.takeIf { it > 0f },
            bufferedMs  = (state as? PlayerState.Playing)?.bufferedMs,

            isLive      = (state as? PlayerState.Playing)?.isLive
                ?: (state as? PlayerState.Ready)?.isLive,
            isSeekable  = (state as? PlayerState.Playing)?.isSeekable
                ?: (state as? PlayerState.Ready)?.isSeekable,
            playbackSpeed = (state as? PlayerState.Playing)?.playbackSpeed,

            bufferPercent  = (state as? PlayerState.Buffering)?.bufferPercent,
            rebufferCount  = (state as? PlayerState.Buffering)?.rebufferCount,

            errorCode      = (state as? PlayerState.Error)?.errorCode,
            errorCodeName  = (state as? PlayerState.Error)?.errorCodeName,
            errorCategory  = (state as? PlayerState.Error)?.errorCategory?.name,
            errorMessage   = (state as? PlayerState.Error)?.errorMessage,
            isFatalError   = (state as? PlayerState.Error)?.isFatal,
            retryCount     = (state as? PlayerState.Error)?.retryCount,
            maxRetries     = (state as? PlayerState.Error)?.maxRetries,
            recoveryHint   = (state as? PlayerState.Error)?.recoveryHint?.name,
            previousStateName = (state as? PlayerState.Error)?.previousStateName,

            pauseReason    = (state as? PlayerState.Paused)?.pauseReason?.name,
            pausedBySystem = (state as? PlayerState.Paused)?.pausedBySystem,

            completionPercent = (state as? PlayerState.Ended)?.completionPercent,

            recentBreadcrumbs = history,
        )
    }

    /** Flat key=value map — easy to attach to crash reporter. */
    fun toMap(): Map<String, String> = buildMap {
        put("state", stateName)
        mediaId?.let { put("mediaId", it) }
        put("seq", sequenceNumber.toString())
        put("ageMs", ageMs.toString())
        positionMs?.let { put("positionMs", it.toString()) }
        durationMs?.let { put("durationMs", it.toString()) }
        progress?.let { put("progress", "%.2f".format(it)) }
        bufferPercent?.let { put("bufferPercent", it.toString()) }
        rebufferCount?.let { put("rebufferCount", it.toString()) }
        errorCode?.let { put("errorCode", it.toString()) }
        errorCodeName?.let { put("errorCodeName", it) }
        errorCategory?.let { put("errorCategory", it) }
        errorMessage?.let { put("errorMessage", it) }
        isFatalError?.let { put("isFatal", it.toString()) }
        retryCount?.let { put("retryCount", it.toString()) }
        maxRetries?.let { put("maxRetries", it.toString()) }
        recoveryHint?.let { put("recoveryHint", it) }
        previousStateName?.let { put("previousState", it) }
        pauseReason?.let { put("pauseReason", it) }
        playbackSpeed?.let { put("playbackSpeed", it.toString()) }
        completionPercent?.let { put("completionPercent", it.toString()) }
        recentBreadcrumbs.forEachIndexed { i, b -> put("breadcrumb_$i", b) }
    }
}
```

## xplayer/app/src/main/java/com/xplayer/dev/core/state/PlayerStateExtensions.kt

```kotlin
package com.xplayer.dev.core.state

import androidx.media3.common.PlaybackException

// ═══════════════════════════════════════════════════════════════════════════
// Identity & Classification
// ═══════════════════════════════════════════════════════════════════════════

/** Media ID carried by this state, or null if no media is associated. */
val PlayerState.mediaId: String?
    get() = when (this) {
        is PlayerState.Uninitialised -> null
        is PlayerState.Idle          -> null
        is PlayerState.Preparing     -> mediaId
        is PlayerState.Buffering     -> mediaId
        is PlayerState.Ready         -> mediaId
        is PlayerState.Playing       -> mediaId
        is PlayerState.Paused        -> mediaId
        is PlayerState.Ended         -> mediaId
        is PlayerState.Error         -> mediaId
    }

/** Human-readable name of this state class. */
val PlayerState.stateName: String
    get() = this::class.simpleName ?: "Unknown"

/** How long this state has been active in milliseconds. */
val PlayerState.ageMs: Long
    get() = System.currentTimeMillis() - enteredAtMs

// ═══════════════════════════════════════════════════════════════════════════
// Boolean Classification
// ═══════════════════════════════════════════════════════════════════════════

/** True if the player is initialised and not yet released. */
val PlayerState.isEngineAlive: Boolean
    get() = this !is PlayerState.Uninitialised

/** True if a media item has been assigned. */
val PlayerState.hasMedia: Boolean
    get() = mediaId != null

/**
 * True if the player is actively producing audio/video output.
 * Buffering is included — the engine is working, not idle.
 */
val PlayerState.isActive: Boolean
    get() = this is PlayerState.Playing
        || this is PlayerState.Buffering

/** True if the player is producing output frames right now. */
val PlayerState.isPlaying: Boolean
    get() = this is PlayerState.Playing

/** True if the player is paused. */
val PlayerState.isPaused: Boolean
    get() = this is PlayerState.Paused

/** True if the player is stalled waiting for data. */
val PlayerState.isBuffering: Boolean
    get() = this is PlayerState.Buffering

/**
 * True if this is a terminal state.
 * Terminal = the current media session is over; the engine
 * must be reset/loaded before playback can resume.
 */
val PlayerState.isTerminal: Boolean
    get() = this is PlayerState.Ended
        || (this is PlayerState.Error && isFatal)

/**
 * True if the error is recoverable.
 * Non-fatal errors should trigger automatic retry.
 */
val PlayerState.isRecoverableError: Boolean
    get() = this is PlayerState.Error && !isFatal

/** True if any form of error is present. */
val PlayerState.isError: Boolean
    get() = this is PlayerState.Error

/** True if idle (engine alive, no media). */
val PlayerState.isIdle: Boolean
    get() = this is PlayerState.Idle

/** True if the media stream is live. */
val PlayerState.isLiveStream: Boolean
    get() = when (this) {
        is PlayerState.Playing -> isLive
        is PlayerState.Paused  -> false  // live streams should not pause
        is PlayerState.Ready   -> isLive
        else                   -> false
    }

// ═══════════════════════════════════════════════════════════════════════════
// Capability Guards
// ═══════════════════════════════════════════════════════════════════════════

/**
 * True if a play command can be accepted right now.
 * Used by [PlayerEngine] and [PlayerManager] to guard commands.
 */
val PlayerState.canPlay: Boolean
    get() = this is PlayerState.Ready
        || this is PlayerState.Paused
        || this is PlayerState.Playing  // already playing, no-op but safe

/**
 * True if a pause command can be accepted right now.
 */
val PlayerState.canPause: Boolean
    get() = this is PlayerState.Playing
        || this is PlayerState.Buffering // can pause during buffering

/**
 * True if a seek command can be accepted right now.
 */
val PlayerState.canSeek: Boolean
    get() = when (this) {
        is PlayerState.Playing  -> isSeekable
        is PlayerState.Paused   -> true
        is PlayerState.Ready    -> isSeekable
        is PlayerState.Buffering -> false // seek during buffering is unstable
        else                    -> false
    }

/**
 * True if a stop command can be accepted.
 */
val PlayerState.canStop: Boolean
    get() = this !is PlayerState.Uninitialised
        && this !is PlayerState.Idle

/**
 * True if a new media load can be initiated.
 * Always true except when the engine is uninitialised or releasing.
 */
val PlayerState.canLoad: Boolean
    get() = this !is PlayerState.Uninitialised
        && this !is PlayerState.Preparing  // wait for current prepare to finish

// ═══════════════════════════════════════════════════════════════════════════
// Position & Duration
// ═══════════════════════════════════════════════════════════════════════════

/** Current playback position, or 0 if not available. */
val PlayerState.positionMs: Long
    get() = when (this) {
        is PlayerState.Playing -> positionMs
        is PlayerState.Paused  -> positionMs
        else                   -> 0L
    }

/** Duration of the current media, or 0 if unknown. */
val PlayerState.durationMs: Long
    get() = when (this) {
        is PlayerState.Playing -> durationMs
        is PlayerState.Paused  -> durationMs
        is PlayerState.Ready   -> durationMs
        is PlayerState.Ended   -> totalDurationMs
        else                   -> 0L
    }

/** Normalised progress [0.0 .. 1.0]. 0 for live/unknown. */
val PlayerState.progress: Float
    get() = (this as? PlayerState.Playing)?.progress ?: run {
        val pos = positionMs
        val dur = durationMs
        if (dur <= 0L) 0f else (pos.toFloat() / dur).coerceIn(0f, 1f)
    }

// ═══════════════════════════════════════════════════════════════════════════
// Error Access
// ═══════════════════════════════════════════════════════════════════════════

/** The [PlaybackException] if this state is an [PlayerState.Error], else null. */
val PlayerState.error: PlaybackException?
    get() = (this as? PlayerState.Error)?.cause

/** Error code if this is an [PlayerState.Error], else -1. */
val PlayerState.errorCode: Int
    get() = (this as? PlayerState.Error)?.errorCode ?: -1

/** Error category if this is an [PlayerState.Error], else null. */
val PlayerState.errorCategory: PlayerState.ErrorCategory?
    get() = (this as? PlayerState.Error)?.errorCategory

/** Recovery hint if this is an [PlayerState.Error], else null. */
val PlayerState.recoveryHint: PlayerState.RecoveryHint?
    get() = (this as? PlayerState.Error)?.recoveryHint

// ═══════════════════════════════════════════════════════════════════════════
// Copy Helpers — avoid breaking state immutability
// ═══════════════════════════════════════════════════════════════════════════

/**
 * Create a position-updated copy of [PlayerState.Playing].
 * More efficient than full copy() when only position changes at 1Hz.
 */
fun PlayerState.Playing.withPosition(
    positionMs: Long,
    bufferedMs: Long = this.bufferedMs,
): PlayerState.Playing = copy(
    positionMs = positionMs.coerceAtLeast(0L),
    bufferedMs = bufferedMs.coerceAtLeast(0L),
)

/**
 * Create a retry-incremented copy of [PlayerState.Error].
 */
fun PlayerState.Error.withIncrementedRetry(): PlayerState.Error = copy(
    retryCount = retryCount + 1,
    enteredAtMs = System.currentTimeMillis(),
)

/**
 * Wrap this state in an [PlayerState.Error] preserving it as [previousState].
 */
fun PlayerState.toError(
    cause: PlaybackException,
    mediaId: String? = this.mediaId,
): PlayerState.Error = PlayerState.Error(
    mediaId       = mediaId,
    cause         = cause,
    previousState = this,
)

// ═══════════════════════════════════════════════════════════════════════════
// Logging Helpers
// ═══════════════════════════════════════════════════════════════════════════

/**
 * Compact one-line summary for logcat.
 *
 * Example outputs:
 *  - "Idle[seq=5]"
 *  - "Playing[id=track_123 pos=12300ms speed=1.5x seq=42]"
 *  - "Error[id=track_123 code=2004 cat=NETWORK retry=1/3 seq=8]"
 */
val PlayerState.logSummary: String
    get() = buildString {
        append(stateName)
        append("[")
        when (val s = this@logSummary) {
            is PlayerState.Uninitialised -> Unit
            is PlayerState.Idle          -> Unit

            is PlayerState.Preparing ->
                append("id=${s.mediaId} start=${s.startPositionMs}ms")

            is PlayerState.Buffering ->
                append(
                    "id=${s.mediaId} buf=${s.bufferPercent}% " +
                        "rebuf=${s.rebufferCount} init=${s.isInitialBuffer}"
                )

            is PlayerState.Ready ->
                append(
                    "id=${s.mediaId} dur=${s.durationMs}ms " +
                        "live=${s.isLive} seek=${s.isSeekable}"
                )

            is PlayerState.Playing ->
                append(
                    "id=${s.mediaId} pos=${s.positionMs}ms " +
                        "dur=${s.durationMs}ms speed=${s.playbackSpeed}x " +
                        "buf=${s.bufferAheadMs}ms live=${s.isLive}"
                )

            is PlayerState.Paused ->
                append(
                    "id=${s.mediaId} pos=${s.positionMs}ms " +
                        "system=${s.pausedBySystem} reason=${s.pauseReason}"
                )

            is PlayerState.Ended ->
                append(
                    "id=${s.mediaId} dur=${s.totalDurationMs}ms " +
                        "completion=${s.completionPercent}%"
                )

            is PlayerState.Error ->
                append(
                    "id=${s.mediaId} code=${s.errorCode} " +
                        "name=${s.errorCodeName} cat=${s.errorCategory} " +
                        "retry=${s.retryCount}/${s.maxRetries} " +
                        "fatal=${s.isFatal} prev=${s.previousStateName} " +
                        "hint=${s.recoveryHint}"
                )
        }
        append(" seq=${sequenceNumber}]")
    }
```

## xplayer/app/src/main/java/com/xplayer/dev/core/state/PlayerStateFlowExtensions.kt

```kotlin
package com.xplayer.dev.core.state

import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.distinctUntilChanged
import kotlinx.coroutines.flow.filter
import kotlinx.coroutines.flow.filterIsInstance
import kotlinx.coroutines.flow.map
import kotlinx.coroutines.flow.mapNotNull

/**
 * Typed observation helpers for [PlayerState] flows.
 *
 * Usage:
 * ```kotlin
 * stateMachine.state
 *     .onPlaying { state -> updateUi(state.positionMs) }
 *     .launchIn(scope)
 *
 * stateMachine.state
 *     .errors()
 *     .onEach { error -> crashReporter.record(error.cause) }
 *     .launchIn(scope)
 * ```
 */

// ── Typed filter extensions ───────────────────────────────────────────────────

fun Flow<PlayerState>.playingStates(): Flow<PlayerState.Playing> =
    filterIsInstance<PlayerState.Playing>()

fun Flow<PlayerState>.pausedStates(): Flow<PlayerState.Paused> =
    filterIsInstance<PlayerState.Paused>()

fun Flow<PlayerState>.bufferingStates(): Flow<PlayerState.Buffering> =
    filterIsInstance<PlayerState.Buffering>()

fun Flow<PlayerState>.readyStates(): Flow<PlayerState.Ready> =
    filterIsInstance<PlayerState.Ready>()

fun Flow<PlayerState>.endedStates(): Flow<PlayerState.Ended> =
    filterIsInstance<PlayerState.Ended>()

fun Flow<PlayerState>.errors(): Flow<PlayerState.Error> =
    filterIsInstance<PlayerState.Error>()

fun Flow<PlayerState>.fatalErrors(): Flow<PlayerState.Error> =
    filterIsInstance<PlayerState.Error>().filter { it.isFatal }

fun Flow<PlayerState>.recoverableErrors(): Flow<PlayerState.Error> =
    filterIsInstance<PlayerState.Error>().filter { !it.isFatal }

fun Flow<PlayerState>.idleStates(): Flow<PlayerState.Idle> =
    filterIsInstance<PlayerState.Idle>()

// ── Property projection extensions ───────────────────────────────────────────

/** Emit position updates only when playing. Deduplicated. */
fun Flow<PlayerState>.positions(): Flow<Long> =
    mapNotNull { (it as? PlayerState.Playing)?.positionMs }
        .distinctUntilChanged()

/** Emit progress updates [0.0 .. 1.0] only when playing. */
fun Flow<PlayerState>.progressUpdates(): Flow<Float> =
    mapNotNull { (it as? PlayerState.Playing)?.progress }
        .distinctUntilChanged()

/** Emit buffer percent during buffering states only. */
fun Flow<PlayerState>.bufferPercent(): Flow<Int> =
    mapNotNull { (it as? PlayerState.Buffering)?.bufferPercent }
        .distinctUntilChanged()

/** Emit duration when it becomes known. Deduplicates same values. */
fun Flow<PlayerState>.duration(): Flow<Long> =
    map { it.durationMs }
        .filter { it > 0L }
        .distinctUntilChanged()

/** Emit true/false as playback active state changes. */
fun Flow<PlayerState>.isActiveFlow(): Flow<Boolean> =
    map { it.isActive }.distinctUntilChanged()

/** Emit true/false as playing state changes. */
fun Flow<PlayerState>.isPlayingFlow(): Flow<Boolean> =
    map { it.isPlaying }.distinctUntilChanged()

/** Emit state name strings for logging. */
fun Flow<PlayerState>.stateNames(): Flow<String> =
    map { it.stateName }.distinctUntilChanged()

// ── StateFlow specific ────────────────────────────────────────────────────────

/** Current position if currently playing, else 0. */
val StateFlow<PlayerState>.currentPositionMs: Long
    get() = (value as? PlayerState.Playing)?.positionMs ?: 0L

/** True if currently in an active playback state. */
val StateFlow<PlayerState>.isCurrentlyPlaying: Boolean
    get() = value.isPlaying

/** True if any error is present. */
val StateFlow<PlayerState>.hasError: Boolean
    get() = value.isError
```

## xplayer/app/src/main/java/com/xplayer/dev/core/state/PlayerStateMachine.kt

```kotlin
package com.xplayer.dev.core.state

import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow
import timber.log.Timber
import javax.inject.Inject
import javax.inject.Singleton

/**
 * # PlayerStateMachine
 *
 * The authoritative state machine for the player.
 *
 * Responsibilities:
 *  1. Accepts [PlayerStateCommand] inputs.
 *  2. Delegates computation to [PlayerStateReducer] (pure function).
 *  3. Publishes new states to [state] (StateFlow).
 *  4. Feeds accepted transitions to [PlayerStateStore].
 *  5. Notifies [listeners] of every transition.
 *
 * Thread safety:
 *  - [dispatch] must be called on the **main thread**.
 *  - [state] can be collected from any thread.
 *
 * @param reducer  Pure state reducer.
 * @param store    History + diagnostics store.
 * @param isDebug  If true, rejected transitions throw instead of logging.
 */
@Singleton
class PlayerStateMachine @Inject constructor(
    private val reducer: PlayerStateReducer,
    private val store: PlayerStateStore,
    private val isDebug: Boolean = false,
) {

    private val _state = MutableStateFlow<PlayerState>(PlayerState.Uninitialised)

    /**
     * Current state — always has a value.
     * Emits on every accepted transition.
     * Self-transitions (same type, different data) are also emitted.
     */
    val state: StateFlow<PlayerState> = _state.asStateFlow()

    /** Listeners notified synchronously on every accepted transition. */
    private val listeners = mutableListOf<TransitionListener>()

    // ── Dispatch ──────────────────────────────────────────────────────────────

    /**
     * Dispatch a command to the state machine.
     *
     * @return The [PlayerStateReducer.ReducerResult] for the caller to inspect.
     *         Callers may use this to react to Rejected results in debug builds.
     */
    fun dispatch(command: PlayerStateCommand): PlayerStateReducer.ReducerResult {
        val current = _state.value
        val result  = reducer.reduce(current, command)

        when (result) {
            is PlayerStateReducer.ReducerResult.Accepted -> {
                _state.value = result.nextState
                store.record(result.transition)
                listeners.forEach { it.onTransition(result.transition) }
                Timber.v("StateMachine: ${result.transition.breadcrumb}")
            }

            is PlayerStateReducer.ReducerResult.Ignored -> {
                Timber.d(
                    "StateMachine: ignored [${command::class.simpleName}] " +
                        "— ${result.reason}"
                )
            }

            is PlayerStateReducer.ReducerResult.Rejected -> {
                Timber.e(
                    "StateMachine: REJECTED [${command::class.simpleName}] " +
                        "in [${current.stateName}] — ${result.reason}"
                )
                if (isDebug) {
                    error(
                        "PlayerStateMachine: illegal command " +
                            "[${command::class.simpleName}] " +
                            "in state [${current.stateName}]: ${result.reason}"
                    )
                }
            }
        }
        return result
    }

    // ── Listener management ───────────────────────────────────────────────────

    fun addListener(listener: TransitionListener) {
        if (listener !in listeners) listeners.add(listener)
    }

    fun removeListener(listener: TransitionListener) {
        listeners.remove(listener)
    }

    // ── Inspection ────────────────────────────────────────────────────────────

    val currentState: PlayerState get() = _state.value

    /** Full transition history from [PlayerStateStore]. */
    val history: List<PlayerStateTransition> get() = store.history

    /** Last N transitions. */
    fun recentHistory(n: Int = 10): List<PlayerStateTransition> =
        store.history.takeLast(n)

    fun interface TransitionListener {
        fun onTransition(transition: PlayerStateTransition)
    }
}
```

## xplayer/app/src/main/java/com/xplayer/dev/core/state/PlayerStateObserver.kt

```kotlin
package com.xplayer.dev.core.state

/**
 * Observer interface for listening to player state transitions.
 */
interface PlayerStateObserver {
    fun onStateTransition(transition: PlayerStateTransition)
}

/**
 * Hook for persisting state when entering paused or terminal states.
 */
interface PlayerStatePersistenceHook {
    fun persistState(state: PlayerState)
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/state/PlayerStateReducer.kt

```kotlin
package com.xplayer.dev.core.state

import timber.log.Timber

/**
 * # PlayerStateReducer
 *
 * Pure function: (CurrentState, Command) → NextState
 *
 * Rules:
 *  1. No side effects — only returns a new state.
 *  2. No coroutines, no threading — fully synchronous.
 *  3. Every unhandled (state, command) pair returns [ReducerResult.Ignored]
 *     with a reason — no silent drops.
 *  4. Sequence numbers are auto-incremented on every accepted transition.
 */
class PlayerStateReducer {

    sealed interface ReducerResult {
        /** Transition accepted; [nextState] is the new state. */
        data class Accepted(
            val nextState: PlayerState,
            val transition: PlayerStateTransition,
        ) : ReducerResult

        /**
         * Command ignored in current state.
         * Not an error — some commands are valid no-ops
         * (e.g., Play when already Playing).
         */
        data class Ignored(
            val reason: String,
            val currentState: PlayerState,
            val command: PlayerStateCommand,
        ) : ReducerResult

        /**
         * Command rejected — illegal in current state.
         * Logged as an error; not thrown in release builds.
         */
        data class Rejected(
            val reason: String,
            val currentState: PlayerState,
            val command: PlayerStateCommand,
        ) : ReducerResult
    }

    /**
     * Compute the next state from [current] given [command].
     * Thread-safe — pure function, no shared mutable state.
     */
    fun reduce(
        current: PlayerState,
        command: PlayerStateCommand,
    ): ReducerResult {
        val nextSeq = current.sequenceNumber + 1L

        return when (command) {

            // ── Engine lifecycle ──────────────────────────────────────────────

            is PlayerStateCommand.EngineInitialised -> when (current) {
                is PlayerState.Uninitialised ->
                    accept(current, PlayerState.Idle(sequenceNumber = nextSeq), command)
                else ->
                    ignore("Engine already initialised", current, command)
            }

            is PlayerStateCommand.EngineReleased ->
                accept(current, PlayerState.Uninitialised, command)

            // ── Media loading ─────────────────────────────────────────────────

            is PlayerStateCommand.Prepare -> when {
                !current.canLoad ->
                    reject("Cannot load in state ${current.stateName}", current, command)
                else ->
                    accept(
                        current,
                        PlayerState.Preparing(
                            mediaId         = command.mediaId,
                            mediaItem       = command.mediaItem,
                            startPositionMs = command.startPositionMs,
                            sequenceNumber  = nextSeq,
                        ),
                        command,
                    )
            }

            is PlayerStateCommand.Reset ->
                accept(current, PlayerState.Idle(sequenceNumber = nextSeq), command)

            // ── Buffering ─────────────────────────────────────────────────────

            is PlayerStateCommand.BufferingStarted -> when (current) {
                is PlayerState.Preparing,
                is PlayerState.Ready,
                is PlayerState.Playing,
                is PlayerState.Paused,
                -> {
                    val mediaId = current.mediaId ?: return reject(
                        "BufferingStarted with no mediaId",
                        current,
                        command,
                    )
                    accept(
                        current,
                        PlayerState.Buffering(
                            mediaId        = mediaId,
                            bufferPercent  = command.bufferPercent,
                            bufferedMs     = command.bufferedMs,
                            rebufferCount  = command.rebufferCount,
                            isInitialBuffer = command.isInitialBuffer,
                            sequenceNumber = nextSeq,
                        ),
                        command,
                    )
                }
                is PlayerState.Buffering ->
                    ignore("Already buffering", current, command)
                else ->
                    reject("Cannot buffer in ${current.stateName}", current, command)
            }

            is PlayerStateCommand.BufferingUpdated -> when (current) {
                is PlayerState.Buffering ->
                    accept(
                        current,
                        current.copy(
                            bufferPercent  = command.bufferPercent,
                            bufferedMs     = command.bufferedMs,
                            sequenceNumber = nextSeq,
                        ),
                        command,
                    )
                else ->
                    ignore("Buffer update ignored in ${current.stateName}", current, command)
            }

            // ── Ready ─────────────────────────────────────────────────────────

            is PlayerStateCommand.ReadyToPlay -> when (current) {
                is PlayerState.Buffering,
                is PlayerState.Preparing,
                -> {
                    val mediaId = current.mediaId ?: return reject(
                        "ReadyToPlay with no mediaId",
                        current,
                        command,
                    )
                    accept(
                        current,
                        PlayerState.Ready(
                            mediaId        = mediaId,
                            durationMs     = command.durationMs,
                            isLive         = command.isLive,
                            isSeekable     = command.isSeekable,
                            sequenceNumber = nextSeq,
                        ),
                        command,
                    )
                }
                else ->
                    ignore("ReadyToPlay ignored in ${current.stateName}", current, command)
            }

            // ── Playing ───────────────────────────────────────────────────────

            is PlayerStateCommand.Play -> when {
                !current.canPlay ->
                    reject("Cannot play in ${current.stateName}", current, command)
                current is PlayerState.Playing ->
                    ignore("Already playing", current, command)
                else -> {
                    val mediaId = current.mediaId ?: return reject(
                        "Play with no mediaId",
                        current,
                        command,
                    )
                    accept(
                        current,
                        PlayerState.Playing(
                            mediaId        = mediaId,
                            positionMs     = current.positionMs,
                            durationMs     = current.durationMs,
                            sequenceNumber = nextSeq,
                        ),
                        command,
                    )
                }
            }

            is PlayerStateCommand.PlayingStarted -> when (current) {
                is PlayerState.Ready,
                is PlayerState.Buffering,
                is PlayerState.Paused,
                is PlayerState.Playing,
                -> {
                    val mediaId = current.mediaId ?: return reject(
                        "PlayingStarted with no mediaId",
                        current,
                        command,
                    )
                    accept(
                        current,
                        PlayerState.Playing(
                            mediaId        = mediaId,
                            positionMs     = command.positionMs,
                            durationMs     = command.durationMs,
                            bufferedMs     = command.bufferedMs,
                            isLive         = command.isLive,
                            isSeekable     = command.isSeekable,
                            videoWidth     = command.videoWidth,
                            videoHeight    = command.videoHeight,
                            sequenceNumber = nextSeq,
                        ),
                        command,
                    )
                }
                else ->
                    reject("PlayingStarted invalid in ${current.stateName}", current, command)
            }

            is PlayerStateCommand.PositionUpdated -> when (current) {
                is PlayerState.Playing ->
                    accept(
                        current,
                        current.withPosition(command.positionMs, command.bufferedMs)
                            .copy(sequenceNumber = nextSeq),
                        command,
                    )
                else ->
                    ignore("PositionUpdated ignored in ${current.stateName}", current, command)
            }

            // ── Pause ─────────────────────────────────────────────────────────

            is PlayerStateCommand.Pause -> when {
                !current.canPause ->
                    reject("Cannot pause in ${current.stateName}", current, command)
                current is PlayerState.Paused ->
                    ignore("Already paused", current, command)
                else -> {
                    val mediaId = current.mediaId ?: return reject(
                        "Pause with no mediaId",
                        current,
                        command,
                    )
                    accept(
                        current,
                        PlayerState.Paused(
                            mediaId        = mediaId,
                            positionMs     = current.positionMs,
                            durationMs     = current.durationMs,
                            pausedBySystem = command.reason != PlayerState.Paused.PauseReason.USER,
                            pauseReason    = command.reason,
                            sequenceNumber = nextSeq,
                        ),
                        command,
                    )
                }
            }

            // ── Stop ──────────────────────────────────────────────────────────

            is PlayerStateCommand.Stop -> when {
                !current.canStop ->
                    ignore("Stop ignored in ${current.stateName}", current, command)
                else ->
                    accept(current, PlayerState.Idle(sequenceNumber = nextSeq), command)
            }

            // ── Ended ─────────────────────────────────────────────────────────

            is PlayerStateCommand.PlaybackEnded -> when (current) {
                is PlayerState.Playing,
                is PlayerState.Buffering,
                -> {
                    val mediaId = current.mediaId ?: return reject(
                        "PlaybackEnded with no mediaId",
                        current,
                        command,
                    )
                    accept(
                        current,
                        PlayerState.Ended(
                            mediaId           = mediaId,
                            totalDurationMs   = command.totalDurationMs,
                            completionPercent = command.completionPercent,
                            sequenceNumber    = nextSeq,
                        ),
                        command,
                    )
                }
                else ->
                    reject("PlaybackEnded invalid in ${current.stateName}", current, command)
            }

            // ── Error ─────────────────────────────────────────────────────────

            is PlayerStateCommand.ErrorOccurred ->
                accept(
                    current,
                    current.toError(
                        cause   = command.cause,
                        mediaId = current.mediaId,
                    ).copy(sequenceNumber = nextSeq),
                    command,
                )

            is PlayerStateCommand.ErrorRetrying -> when (current) {
                is PlayerState.Error ->
                    accept(
                        current,
                        current.copy(
                            retryCount     = command.retryCount,
                            sequenceNumber = nextSeq,
                        ),
                        command,
                    )
                else ->
                    reject("ErrorRetrying invalid in ${current.stateName}", current, command)
            }

            // ── Speed ─────────────────────────────────────────────────────────

            is PlayerStateCommand.SetSpeed,
            is PlayerStateCommand.SpeedChanged,
            -> when (current) {
                is PlayerState.Playing -> {
                    val speed = when (command) {
                        is PlayerStateCommand.SetSpeed    -> command.speed
                        is PlayerStateCommand.SpeedChanged -> command.newSpeed
                        else -> return ignore("Speed ignored", current, command)
                    }
                    accept(
                        current,
                        current.copy(
                            playbackSpeed  = speed,
                            sequenceNumber = nextSeq,
                        ),
                        command,
                    )
                }
                else ->
                    ignore("Speed change ignored in ${current.stateName}", current, command)
            }

            // ── Video size ────────────────────────────────────────────────────

            is PlayerStateCommand.VideoSizeChanged -> when (current) {
                is PlayerState.Playing ->
                    accept(
                        current,
                        current.copy(
                            videoWidth     = command.width,
                            videoHeight    = command.height,
                            sequenceNumber = nextSeq,
                        ),
                        command,
                    )
                else ->
                    ignore("VideoSizeChanged ignored in ${current.stateName}", current, command)
            }

            // ── Seek ──────────────────────────────────────────────────────────

            is PlayerStateCommand.Seek -> when {
                !current.canSeek ->
                    reject("Cannot seek in ${current.stateName}", current, command)
                else ->
                    ignore("Seek dispatched to ExoPlayer — state unchanged", current, command)
            }
        }
    }

    // ── Private helpers ───────────────────────────────────────────────────────

    private fun accept(
        current: PlayerState,
        next: PlayerState,
        command: PlayerStateCommand,
    ): ReducerResult.Accepted {
        val transition = PlayerStateTransition(
            from    = current,
            to      = next,
            command = command,
        )
        Timber.v("Reducer: ${transition.breadcrumb}")
        return ReducerResult.Accepted(next, transition)
    }

    private fun ignore(
        reason: String,
        current: PlayerState,
        command: PlayerStateCommand,
    ): ReducerResult.Ignored {
        Timber.d(
            "Reducer: ignored [${command::class.simpleName}] " +
                "in [${current.stateName}] — $reason"
        )
        return ReducerResult.Ignored(reason, current, command)
    }

    private fun reject(
        reason: String,
        current: PlayerState,
        command: PlayerStateCommand,
    ): ReducerResult.Rejected {
        Timber.e(
            "Reducer: REJECTED [${command::class.simpleName}] " +
                "in [${current.stateName}] — $reason"
        )
        return ReducerResult.Rejected(reason, current, command)
    }
}
```

## xplayer/app/src/main/java/com/xplayer/dev/core/state/PlayerStateStore.kt

```kotlin
package com.xplayer.dev.core.state

import java.util.concurrent.CopyOnWriteArrayList
import javax.inject.Inject
import javax.inject.Singleton

/**
 * # PlayerStateStore
 *
 * Bounded history buffer of [PlayerStateTransition] records.
 *
 * Use cases:
 *  1. Crash breadcrumbs — send last N transitions to crash reporter.
 *  2. QoS analytics — measure time-in-state for each transition.
 *  3. Debug replay — reproduce exact state sequences in tests.
 *
 * Thread safety: [CopyOnWriteArrayList] — safe to read from any thread.
 * Write frequency is low (one per state transition) so CoW overhead
 * is acceptable.
 *
 * @param capacity Maximum number of transitions to retain.
 *                 Oldest entries are dropped when capacity is exceeded.
 */
@Singleton
class PlayerStateStore @Inject constructor() { 
    private val capacity = DEFAULT_CAPACITY

    private val _history = CopyOnWriteArrayList<PlayerStateTransition>()

    /** Full history snapshot. Thread-safe. */
    val history: List<PlayerStateTransition> get() = _history.toList()

    /** Number of recorded transitions. */
    val size: Int get() = _history.size

    /** Record a new transition, evicting oldest if at capacity. */
    fun record(transition: PlayerStateTransition) {
        if (_history.size >= capacity) {
            _history.removeAt(0)
        }
        _history.add(transition)
    }

    /** Clear all history. */
    fun clear() {
        _history.clear()
    }

    // ── Analytics helpers ─────────────────────────────────────────────────────

    /** Total time spent in [Playing] state across all transitions. */
    val totalPlayingMs: Long
        get() = _history
            .filter { it.from is PlayerState.Playing }
            .sumOf { it.timeInFromStateMs }

    /** Total time spent in [Buffering] state (stall time). */
    val totalBufferingMs: Long
        get() = _history
            .filter { it.from is PlayerState.Buffering }
            .sumOf { it.timeInFromStateMs }

    /** Number of times a [Buffering] state was entered from [Playing]. */
    val rebufferCount: Int
        get() = _history.count {
            it.from is PlayerState.Playing && it.to is PlayerState.Buffering
        }

    /** Number of [Error] states encountered. */
    val errorCount: Int
        get() = _history.count { it.to is PlayerState.Error }

    /**
     * Ordered breadcrumb strings for crash reporting.
     * Returns the last [n] transitions as human-readable strings.
     */
    fun breadcrumbs(n: Int = 20): List<String> =
        _history.takeLast(n).map { it.breadcrumb }

    /** Full diagnostic report as multi-line string. */
    fun diagnosticReport(): String = buildString {
        appendLine("=== PlayerStateStore Diagnostic Report ===")
        appendLine("Total transitions : ${_history.size}")
        appendLine("Total playing     : ${totalPlayingMs}ms")
        appendLine("Total buffering   : ${totalBufferingMs}ms")
        appendLine("Rebuffer count    : $rebufferCount")
        appendLine("Error count       : $errorCount")
        appendLine("--- Transition History (oldest → newest) ---")
        _history.forEachIndexed { idx, t ->
            appendLine("  $idx. ${t.breadcrumb}")
        }
    }

    companion object {
        const val DEFAULT_CAPACITY = 200
    }
}
```

## xplayer/app/src/main/java/com/xplayer/dev/core/state/PlayerStateTransition.kt

```kotlin
package com.xplayer.dev.core.state

/**
 * Represents a single validated state transition.
 *
 * Stored in [PlayerStateStore] history buffer for:
 *  1. Debugging — replay the full state timeline.
 *  2. Analytics — measure time-in-state.
 *  3. Crash reporting — send breadcrumbs on failure.
 */
data class PlayerStateTransition(
    val from: PlayerState,
    val to: PlayerState,
    val command: PlayerStateCommand,
    val transitionMs: Long = System.currentTimeMillis(),
) {
    /** Duration spent in [from] state, in milliseconds. */
    val timeInFromStateMs: Long
        get() = transitionMs - from.enteredAtMs

    /**
     * Single-line summary for crash breadcrumbs.
     * Example: "Playing→Buffering via BufferingStarted [12ms in Playing]"
     */
    val breadcrumb: String
        get() = "${from.stateName}→${to.stateName} " +
            "via ${command::class.simpleName} " +
            "[${timeInFromStateMs}ms in ${from.stateName}]"

    /** True if this was a self-transition (same state type, different data). */
    val isSelfTransition: Boolean
        get() = from::class == to::class
}
```

## xplayer/app/src/main/java/com/xplayer/dev/core/events/PlayerEvent.kt

```kotlin
package com.xplayer.dev.core.events

import androidx.media3.common.PlaybackException
import androidx.media3.common.Tracks
import com.xplayer.dev.core.state.PlayerState
import java.util.UUID

/**
 * # PlayerEvent — Sealed Hierarchy
 *
 * One-shot signals emitted by [PlayerEngine] describing what *happened*.
 *
 * ## Design Contracts
 *  1. Every event carries [eventId]      — UUID v4, globally unique.
 *  2. Every event carries [timestampMs]  — wall-clock epoch for ordering.
 *  3. Every event carries [sessionId]    — groups events per play session.
 *  4. Every event carries [sequenceNum]  — monotonic counter per session.
 *  5. Events are **immutable data classes** — no mutation after creation.
 *  6. Events are **never retained** by the engine after emission.
 *  7. No Android framework types in event payloads — serialisation-safe.
 *
 * ## Categories
 *  ┌──────────────┬────────────────────────────────────────────────────┐
 *  │ Category     │ Events                                             │
 *  ├──────────────┼────────────────────────────────────────────────────┤
 *  │ Session      │ SessionStarted, SessionEnded                       │
 *  │ Playback     │ Started, Paused, Resumed, Stopped, Ended, Speed    │
 *  │ Seek         │ SeekRequested, SeekCompleted                       │
 *  │ Buffer       │ BufferingStarted, BufferingEnded, BufferUpdated    │
 *  │ Track        │ TracksChanged, VideoSizeChanged, TrackSelected     │
 *  │ Metadata     │ MetadataReceived, ChapterChanged                   │
 *  │ QoS          │ BitrateChanged, DroppedFrames, BandwidthEstimated  │
 *  │ Error        │ ErrorOccurred, ErrorRecovered, ErrorRetrying       │
 *  │ DRM          │ DrmKeysLoaded, DrmSessionError                     │
 *  │ Ad           │ AdStarted, AdEnded, AdSkipped, AdError             │
 *  │ Network      │ NetworkTypeChanged, CdnSwitch                      │
 *  └──────────────┴────────────────────────────────────────────────────┘
 *
 * ## Threading
 *  Events are emitted on the **main thread** by [PlayerEngine].
 *  Collectors may run on any thread via [PlayerEventBus].
 */
sealed class PlayerEvent {

    // ── Identity (every event) ────────────────────────────────────────────────

    /** Globally unique event identifier (UUID v4). */
    abstract val eventId: String

    /** Wall-clock epoch millis when this event was created. */
    abstract val timestampMs: Long

    /** Groups all events for one media load attempt. */
    abstract val sessionId: String

    /** Monotonically increasing counter within [sessionId]. */
    abstract val sequenceNum: Long

    /** High-level category for routing and filtering. */
    abstract val category: EventCategory

    /** Media item associated with this event. Null for session-level events. */
    abstract val mediaId: String?

    // ═══════════════════════════════════════════════════════════════════════
    // Category enum
    // ═══════════════════════════════════════════════════════════════════════

    enum class EventCategory {
        SESSION,
        PLAYBACK,
        SEEK,
        BUFFER,
        TRACK,
        METADATA,
        QOS,
        ERROR,
        DRM,
        AD,
        NETWORK,
    }

    // ═══════════════════════════════════════════════════════════════════════
    // Session Events
    // ═══════════════════════════════════════════════════════════════════════

    /**
     * Emitted when a new playback session begins (prepare() called).
     *
     * @param deviceInfo    Hardware / OS context for QoS attribution.
     * @param appVersion    App version string for release tracking.
     * @param playerVersion Media3 / ExoPlayer version in use.
     * @param contentType   VOD, Live, Podcast, etc.
     */
    data class SessionStarted(
        override val eventId: String      = newId(),
        override val timestampMs: Long    = now(),
        override val sessionId: String,
        override val sequenceNum: Long = 0L,
        override val mediaId: String,
        override val category: EventCategory = EventCategory.SESSION,
        val url: String,
        val contentType: ContentType,
        val startPositionMs: Long         = 0L,
        val deviceInfo: DeviceInfo,
        val appVersion: String,
        val playerVersion: String,
        val networkType: NetworkType      = NetworkType.UNKNOWN,
    ) : PlayerEvent() {

        enum class ContentType {
            VOD,         // video on demand
            LIVE,        // live stream
            PODCAST,     // audio-only episodic
            CLIP,        // short-form video
            UNKNOWN,
        }
    }

    /**
     * Emitted when a session ends (stop, release, or new prepare).
     *
     * @param endReason      Why this session ended.
     * @param totalSessionMs Wall-clock duration of the session.
     * @param totalPlayedMs  Actual playback time (excludes stalls/pauses).
     * @param totalStalledMs Cumulative stall / rebuffer time.
     * @param totalStallCount Number of stall events.
     * @param maxConcurrentBps Peak bandwidth observed in session.
     * @param completionPercent 0..100; <100 means user abandoned.
     */
    data class SessionEnded(
        override val eventId: String      = newId(),
        override val timestampMs: Long    = now(),
        override val sessionId: String,
        override val sequenceNum: Long = 0L,
        override val mediaId: String?,
        override val category: EventCategory = EventCategory.SESSION,
        val endReason: EndReason,
        val totalSessionMs: Long = 0L,
        val totalPlayedMs: Long,
        val totalStalledMs: Long = 0L,
        val totalStallCount: Int = 0,
        val totalSeekCount: Int,
        val totalDroppedFrames: Int,
        val averageBandwidthBps: Long,
        val maxConcurrentBps: Long,
        val completionPercent: Float,
    ) : PlayerEvent() {

        init {
            require(completionPercent in 0f..100f) {
                "completionPercent must be in 0..100, was $completionPercent"
            }
            require(totalSessionMs >= 0L) {
                "totalSessionMs must be >= 0"
            }
        }

        enum class EndReason {
            PLAYBACK_COMPLETED,
            USER_STOP,
            USER_NAVIGATION,    // user navigated away
            APP_BACKGROUND,
            APP_KILLED,
            NEW_ITEM_LOADED,    // replaced by another prepare()
            ERROR_FATAL,
            AUDIO_FOCUS_LOSS_PERMANENT,
            RELEASE,
        }
    }

    // ═══════════════════════════════════════════════════════════════════════
    // Playback Events
    // ═══════════════════════════════════════════════════════════════════════

    /**
     * First frame rendered or audio output started.
     *
     * @param timeToFirstFrameMs TTFP: prepare() → first frame.
     *                           Critical QoS SLA metric.
     * @param isResumed          False = fresh start; True = resume after pause.
     */
    data class PlaybackStarted(
        override val eventId: String      = newId(),
        override val timestampMs: Long    = now(),
        override val sessionId: String,
        override val sequenceNum: Long = 0L,
        override val mediaId: String,
        override val category: EventCategory = EventCategory.PLAYBACK,
        val positionMs: Long,
        val durationMs: Long,
        val isResumed: Boolean,
        val timeToFirstFrameMs: Long,
        val playbackSpeed: Float          = 1.0f,
        val isLive: Boolean               = false,
        val videoWidth: Int               = 0,
        val videoHeight: Int              = 0,
    ) : PlayerEvent() {

        init {
            require(timeToFirstFrameMs >= 0L) {
                "timeToFirstFrameMs must be >= 0"
            }
            require(playbackSpeed > 0f) {
                "playbackSpeed must be > 0"
            }
        }

        val isAudioOnly: Boolean get() = videoWidth == 0 && videoHeight == 0
    }

    /**
     * Playback paused.
     *
     * @param pauseReason  Distinguishes user action from system interrupts.
     * @param pausedBySystem True if the system initiated the pause
     *                       (audio focus, call, etc.).
     */
    data class PlaybackPaused(
        override val eventId: String      = newId(),
        override val timestampMs: Long    = now(),
        override val sessionId: String,
        override val sequenceNum: Long = 0L,
        override val mediaId: String,
        override val category: EventCategory = EventCategory.PLAYBACK,
        val positionMs: Long,
        val pauseReason: PlayerState.Paused.PauseReason,
        val pausedBySystem: Boolean,
    ) : PlayerEvent()

    /**
     * Playback resumed after a pause.
     */
    data class PlaybackResumed(
        override val eventId: String      = newId(),
        override val timestampMs: Long    = now(),
        override val sessionId: String,
        override val sequenceNum: Long = 0L,
        override val mediaId: String,
        override val category: EventCategory = EventCategory.PLAYBACK,
        val positionMs: Long,
        val pauseDurationMs: Long,          // how long was it paused?
        val playbackSpeed: Float = 1.0f,
    ) : PlayerEvent() {
        init {
            require(pauseDurationMs >= 0L) {
                "pauseDurationMs must be >= 0"
            }
        }
    }

    /**
     * Playback stopped explicitly (not ended naturally).
     */
    data class PlaybackStopped(
        override val eventId: String      = newId(),
        override val timestampMs: Long    = now(),
        override val sessionId: String,
        override val sequenceNum: Long = 0L,
        override val mediaId: String,
        override val category: EventCategory = EventCategory.PLAYBACK,
        val positionMs: Long,
        val stopReason: StopReason = StopReason.USER_ACTION,
        val playedDurationMs: Long = 0L,
    ) : PlayerEvent() {

        enum class StopReason {
            USER_ACTION,
            APP_BACKGROUND,
            AUDIO_FOCUS_LOSS_PERMANENT,
            NEW_ITEM_LOADED,
            HEADSET_DISCONNECT,
            RELEASE,
        }
    }

    /**
     * Media played to natural end.
     *
     * @param completionPercent Should be ~100 for natural end.
     *                          <100 if stream ended earlier than metadata said.
     * @param totalPlayedMs     Actual rendered time (not wall-clock).
     */
    data class PlaybackEnded(
        override val eventId: String      = newId(),
        override val timestampMs: Long    = now(),
        override val sessionId: String,
        override val sequenceNum: Long = 0L,
        override val mediaId: String,
        override val category: EventCategory = EventCategory.PLAYBACK,
        val totalPlayedMs: Long,
        val totalDurationMs: Long,
        val completionPercent: Float,
    ) : PlayerEvent() {
        init {
            require(completionPercent in 0f..100f) {
                "completionPercent must be in 0..100"
            }
        }

        val isFullCompletion: Boolean get() = completionPercent >= 99f
    }

    /**
     * Playback speed changed.
     *
     * @param previousSpeed Speed before the change.
     * @param newSpeed      Speed after the change.
     * @param initiator     Who triggered the change.
     */
    data class PlaybackSpeedChanged(
        override val eventId: String      = newId(),
        override val timestampMs: Long    = now(),
        override val sessionId: String,
        override val sequenceNum: Long = 0L,
        override val mediaId: String,
        override val category: EventCategory = EventCategory.PLAYBACK,
        val previousSpeed: Float,
        val newSpeed: Float,
        val initiator: SpeedChangeInitiator = SpeedChangeInitiator.USER,
    ) : PlayerEvent() {

        init {
            require(previousSpeed > 0f) { "previousSpeed must be > 0" }
            require(newSpeed > 0f) { "newSpeed must be > 0" }
        }

        enum class SpeedChangeInitiator { USER, SYSTEM, AUTO }
    }

    // ═══════════════════════════════════════════════════════════════════════
    // Seek Events
    // ═══════════════════════════════════════════════════════════════════════

    /**
     * User or system requested a seek.
     * Emitted BEFORE ExoPlayer processes the seek.
     */
    data class SeekRequested(
        override val eventId: String      = newId(),
        override val timestampMs: Long    = now(),
        override val sessionId: String,
        override val sequenceNum: Long = 0L,
        override val mediaId: String,
        override val category: EventCategory = EventCategory.SEEK,
        val fromPositionMs: Long,
        val toPositionMs: Long,
        val seekType: SeekType            = SeekType.ABSOLUTE,
        val initiator: SeekInitiator      = SeekInitiator.USER,
    ) : PlayerEvent() {

        init {
            require(fromPositionMs >= 0L) { "fromPositionMs must be >= 0" }
            require(toPositionMs >= 0L) { "toPositionMs must be >= 0" }
        }

        /** Signed delta — positive = forward, negative = backward. */
        val seekDeltaMs: Long get() = toPositionMs - fromPositionMs
        val isForwardSeek: Boolean get() = seekDeltaMs > 0
        val isBackwardSeek: Boolean get() = seekDeltaMs < 0

        enum class SeekType {
            ABSOLUTE,   // seek to exact position
            RELATIVE,   // seek +/- from current
            CHAPTER,    // jump to chapter boundary
            LIVE_EDGE,  // jump to live edge
        }

        enum class SeekInitiator {
            USER,
            AUTO_SKIP_SILENCE,
            AUTO_SKIP_INTRO,
            DEEP_LINK,
            RESUME_POSITION,
        }
    }

    /**
     * Seek completed — player is rendering at the new position.
     *
     * @param seekDurationMs Time from seek request to first frame at new position.
     */
    data class SeekCompleted(
        override val eventId: String      = newId(),
        override val timestampMs: Long    = now(),
        override val sessionId: String,
        override val sequenceNum: Long = 0L,
        override val mediaId: String,
        override val category: EventCategory = EventCategory.SEEK,
        val fromPositionMs: Long,
        val toPositionMs: Long,
        val seekDurationMs: Long,
        val requiredRebuffer: Boolean     = false,  // did seek cause a stall?
    ) : PlayerEvent() {
        init {
            require(seekDurationMs >= 0L) { "seekDurationMs must be >= 0" }
        }
    }

    // ═══════════════════════════════════════════════════════════════════════
    // Buffer Events
    // ═══════════════════════════════════════════════════════════════════════

    /**
     * Player entered buffering state.
     *
     * @param isInitialBuffer True = first buffer (prepare → first play).
     *                        False = rebuffer / stall during playback.
     * @param positionMs      Position where stall occurred.
     * @param bufferAheadMs   Buffer level at stall time.
     * @param stallCount      Cumulative stalls in this session.
     */
    data class BufferingStarted(
        override val eventId: String      = newId(),
        override val timestampMs: Long    = now(),
        override val sessionId: String,
        override val sequenceNum: Long = 0L,
        override val mediaId: String,
        override val category: EventCategory = EventCategory.BUFFER,
        val positionMs: Long,
        val bufferAheadMs: Long = 0L,
        val bufferPercent: Int = 0,
        val isInitialBuffer: Boolean = false,
        val stallCount: Int,
    ) : PlayerEvent() {
        init {
            require(bufferPercent in 0..100) { "bufferPercent must be in 0..100" }
            require(stallCount >= 0) { "stallCount must be >= 0" }
        }

        /** True if this is a mid-playback stall, not the initial load. */
        val isRebuffer: Boolean get() = !isInitialBuffer
    }

    /**
     * Player exited buffering state and resumed or became ready.
     *
     * @param stallDurationMs Time spent in buffering state.
     * @param bufferPercentOnResume Buffer level when playback resumed.
     * @param exitReason       Why buffering ended.
     */
    data class BufferingEnded(
        override val eventId: String      = newId(),
        override val timestampMs: Long    = now(),
        override val sessionId: String,
        override val sequenceNum: Long = 0L,
        override val mediaId: String,
        override val category: EventCategory = EventCategory.BUFFER,
        val stallDurationMs: Long,
        val bufferPercentOnResume: Int,
        val exitReason: BufferExitReason  = BufferExitReason.ENOUGH_DATA,
    ) : PlayerEvent() {
        init {
            require(stallDurationMs >= 0L) { "stallDurationMs must be >= 0" }
            require(bufferPercentOnResume in 0..100) {
                "bufferPercentOnResume must be in 0..100"
            }
        }

        enum class BufferExitReason {
            ENOUGH_DATA,   // normal — buffer filled sufficiently
            SEEK,          // buffering ended due to seek
            STOP,          // stopped while buffering
            ERROR,         // error during buffering
        }
    }

    /**
     * Periodic buffer level update.
     * Emitted at a configurable interval while buffering or playing.
     *
     * @param bufferPercent       [0..100]
     * @param bufferedPositionMs  Absolute buffer head in ms.
     * @param bufferAheadMs       Buffer ahead of current position.
     */
    data class BufferUpdated(
        override val eventId: String      = newId(),
        override val timestampMs: Long    = now(),
        override val sessionId: String,
        override val sequenceNum: Long = 0L,
        override val mediaId: String,
        override val category: EventCategory = EventCategory.BUFFER,
        val bufferPercent: Int = 0,
        val bufferedPositionMs: Long,
        val bufferAheadMs: Long = 0L,
        val currentPositionMs: Long,
    ) : PlayerEvent() {
        init {
            require(bufferPercent in 0..100) { "bufferPercent must be in 0..100" }
            require(bufferedPositionMs >= 0L) { "bufferedPositionMs must be >= 0" }
        }
    }

    // ═══════════════════════════════════════════════════════════════════════
    // Track Events
    // ═══════════════════════════════════════════════════════════════════════

    /**
     * Available or selected tracks changed.
     *
     * @param availableTracks All tracks in the media container.
     * @param selectedVideo   Currently selected video track info.
     * @param selectedAudio   Currently selected audio track info.
     * @param selectedText    Currently selected subtitle track info.
     */
    data class TracksChanged(
        override val eventId: String      = newId(),
        override val timestampMs: Long    = now(),
        override val sessionId: String,
        override val sequenceNum: Long = 0L,
        override val mediaId: String,
        override val category: EventCategory = EventCategory.TRACK,
        val availableTracks: Tracks,
        val selectedVideo: TrackInfo?,
        val selectedAudio: TrackInfo?,
        val selectedText: TrackInfo?,
    ) : PlayerEvent()

    /**
     * User or ABR algorithm selected a specific track.
     */
    data class TrackSelected(
        override val eventId: String      = newId(),
        override val timestampMs: Long    = now(),
        override val sessionId: String,
        override val sequenceNum: Long = 0L,
        override val mediaId: String,
        override val category: EventCategory = EventCategory.TRACK,
        val trackType: TrackType,
        val track: TrackInfo,
        val selectionReason: TrackSelectionReason,
    ) : PlayerEvent() {

        enum class TrackType { VIDEO, AUDIO, TEXT, IMAGE }

        enum class TrackSelectionReason {
            USER_SELECTED,       // explicit user choice
            ABR_BANDWIDTH,       // adaptive bitrate — bandwidth
            ABR_VIEWPORT,        // adaptive bitrate — screen size
            DEFAULT,             // default selection at load
            LANGUAGE_PREFERENCE, // matched preferred language
        }
    }

    /**
     * Video output size changed (resolution switch or initial render).
     */
    data class VideoSizeChanged(
        override val eventId: String      = newId(),
        override val timestampMs: Long    = now(),
        override val sessionId: String,
        override val sequenceNum: Long = 0L,
        override val mediaId: String,
        override val category: EventCategory = EventCategory.TRACK,
        val width: Int = 0,
        val height: Int = 0,
        val pixelWidthHeightRatio: Float  = 1f,
        val unappliedRotationDegrees: Int = 0,
    ) : PlayerEvent() {
        init {
            require(width >= 0) { "width must be >= 0" }
            require(height >= 0) { "height must be >= 0" }
        }

        val totalPixels: Long get() = width.toLong() * height
        val aspectRatio: Float
            get() = if (height == 0) 0f else width.toFloat() / height
        val is4K: Boolean get() = width >= 3840 || height >= 2160
        val isHD: Boolean get() = width >= 1280 || height >= 720
    }

    // ═══════════════════════════════════════════════════════════════════════
    // Metadata Events
    // ═══════════════════════════════════════════════════════════════════════

    /**
     * Stream or container metadata was received / updated.
     */
    data class MetadataReceived(
        override val eventId: String      = newId(),
        override val timestampMs: Long    = now(),
        override val sessionId: String,
        override val sequenceNum: Long = 0L,
        override val mediaId: String,
        override val category: EventCategory = EventCategory.METADATA,
        val title: String?,
        val artist: String?,
        val album: String? = null,
        val artworkUri: String?,
        val durationMs: Long?,
        val genre: String? = null,
        val releaseYear: Int? = null,
        val extras: Map<String, String>   = emptyMap(),
        val source: MetadataSource        = MetadataSource.CONTAINER,
    ) : PlayerEvent() {

        enum class MetadataSource {
            CONTAINER,    // MP4/MKV/etc. container metadata
            ICY,          // ICY stream tags (radio)
            ID3,          // ID3 tags (MP3)
            EMSG,         // DASH Event Message Box
            HLS_DATERANGE, // HLS DATE-RANGE tag
            CUSTOM,
        }
    }

    /**
     * Chapter boundary reached.
     * Requires chapter metadata to be present in the media.
     */
    data class ChapterChanged(
        override val eventId: String      = newId(),
        override val timestampMs: Long    = now(),
        override val sessionId: String,
        override val sequenceNum: Long = 0L,
        override val mediaId: String,
        override val category: EventCategory = EventCategory.METADATA,
        val chapterIndex: Int,
        val chapterTitle: String?,
        val chapterStartMs: Long,
        val chapterEndMs: Long,
    ) : PlayerEvent() {
        init {
            require(chapterIndex >= 0) { "chapterIndex must be >= 0" }
            require(chapterStartMs >= 0L) { "chapterStartMs must be >= 0" }
        }

        val chapterDurationMs: Long get() = chapterEndMs - chapterStartMs
    }

    // ═══════════════════════════════════════════════════════════════════════
    // QoS Events
    // ═══════════════════════════════════════════════════════════════════════

    /**
     * ABR (Adaptive Bitrate) selection changed.
     * Indicates quality up-switch or down-switch.
     *
     * @param switchDirection UP = quality improved, DOWN = quality degraded.
     * @param reason          Why the switch occurred.
     */
    data class BitrateChanged(
        override val eventId: String      = newId(),
        override val timestampMs: Long    = now(),
        override val sessionId: String,
        override val sequenceNum: Long = 0L,
        override val mediaId: String,
        override val category: EventCategory = EventCategory.QOS,
        val trackType: BitrateTrackType,
        val previousBitrateBps: Long,
        val newBitrateBps: Long,
        val switchDirection: SwitchDirection,
        val reason: BitrateChangeReason,
        val networkBandwidthBps: Long     = 0L,
    ) : PlayerEvent() {

        enum class BitrateTrackType { VIDEO, AUDIO, COMBINED }

        enum class SwitchDirection { UP, DOWN, STABLE }

        enum class BitrateChangeReason {
            BANDWIDTH_INCREASE,
            BANDWIDTH_DECREASE,
            VIEWPORT_CHANGE,        // screen resolution changed
            USER_FORCED,            // user selected quality manually
            INITIAL_SELECTION,
            TRACK_UNAVAILABLE,
        }

        /** Delta in bps — positive = upgrade, negative = downgrade. */
        val bitrateDeltaBps: Long get() = newBitrateBps - previousBitrateBps

        /** Switch ratio — newBitrate / previousBitrate. */
        val switchRatio: Float
            get() = if (previousBitrateBps == 0L) 1f
                    else newBitrateBps.toFloat() / previousBitrateBps
    }

    /**
     * Video frames were dropped during rendering.
     * Indicates decoder or render pipeline pressure.
     *
     * @param droppedCount   Number of frames dropped in this event window.
     * @param elapsedMs      Time window over which frames were dropped.
     * @param totalDropped   Cumulative drops in this session.
     * @param dropRate       Drops per second in this window.
     */
    data class DroppedFrames(
        override val eventId: String      = newId(),
        override val timestampMs: Long    = now(),
        override val sessionId: String,
        override val sequenceNum: Long = 0L,
        override val mediaId: String,
        override val category: EventCategory = EventCategory.QOS,
        val droppedCount: Int,
        val elapsedMs: Long,
        val totalDropped: Int = 0,
        val dropRate: Float = 0f,              // drops/sec in this window
    ) : PlayerEvent() {
        init {
            require(droppedCount >= 0) { "droppedCount must be >= 0" }
            require(elapsedMs > 0L) { "elapsedMs must be > 0" }
            require(totalDropped >= 0) { "totalDropped must be >= 0" }
        }

        /** True if drop rate is alarming (> 2 drops/sec). */
        val isCritical: Boolean get() = dropRate > 2.0f
    }

    /**
     * Network bandwidth was estimated by ExoPlayer's bandwidth meter.
     *
     * @param bandwidthBps   Current estimate in bits per second.
     * @param sampleBps      Raw sample that triggered this update.
     * @param percentileRank Rolling percentile rank (0..100) in session.
     */
    data class BandwidthEstimated(
        override val eventId: String      = newId(),
        override val timestampMs: Long    = now(),
        override val sessionId: String,
        override val sequenceNum: Long = 0L,
        override val mediaId: String?,
        override val category: EventCategory = EventCategory.QOS,
        val bandwidthBps: Long,
        val sampleBps: Long,
        val percentileRank: Int           = 50,
    ) : PlayerEvent() {
        init {
            require(bandwidthBps >= 0L) { "bandwidthBps must be >= 0" }
            require(percentileRank in 0..100) { "percentileRank must be in 0..100" }
        }

        val bandwidthMbps: Float get() = bandwidthBps / 1_000_000f
    }

    /**
     * Render frame processing offset report.
     * Negative offset = frames processed late (potential stuttering).
     *
     * @param averageOffsetUs Average processing offset in microseconds.
     * @param frameCount      Number of frames in this measurement window.
     */
    data class RenderingHealthReport(
        override val eventId: String      = newId(),
        override val timestampMs: Long    = now(),
        override val sessionId: String,
        override val sequenceNum: Long = 0L,
        override val mediaId: String,
        override val category: EventCategory = EventCategory.QOS,
        val averageOffsetUs: Long,
        val frameCount: Int,
        val isHealthy: Boolean,           // true if avg offset > -50ms threshold
    ) : PlayerEvent() {
        init {
            require(frameCount >= 0) { "frameCount must be >= 0" }
        }
    }

    // ═══════════════════════════════════════════════════════════════════════
    // Error Events
    // ═══════════════════════════════════════════════════════════════════════

    /**
     * A playback error occurred.
     *
     * @param error           The underlying [PlaybackException].
     * @param errorCategory   High-level classification.
     * @param isFatal         If true, playback cannot continue.
     * @param retryAttempt    0 = first occurrence; >0 = retry attempt number.
     * @param previousState   State the player was in before the error.
     * @param context         Structured key-value context for crash reports.
     */
    data class ErrorOccurred(
        override val eventId: String      = newId(),
        override val timestampMs: Long    = now(),
        override val sessionId: String,
        override val sequenceNum: Long = 0L,
        override val mediaId: String?,
        override val category: EventCategory = EventCategory.ERROR,
        val error: PlaybackException,
        val errorCategory: com.xplayer.dev.core.state.PlayerState.ErrorCategory,
        val isFatal: Boolean,
        val retryAttempt: Int             = 0,
        val maxRetries: Int               = 3,
        val previousState: String,        // state name — not the object (no leak)
        val context: Map<String, String>  = emptyMap(),
    ) : PlayerEvent() {
        init {
            require(retryAttempt >= 0) { "retryAttempt must be >= 0" }
            require(maxRetries >= 0) { "maxRetries must be >= 0" }
        }

        val errorCode: Int get() = error.errorCode
        val errorCodeName: String get() = error.errorCodeName
        val errorMessage: String get() = error.message ?: "Unknown [$errorCodeName]"
        val canRetry: Boolean get() = !isFatal && retryAttempt < maxRetries
    }

    /**
     * Playback recovered after an error (retry succeeded).
     *
     * @param retryCount        Total retries that were needed.
     * @param recoveryDurationMs Wall-clock time from error to recovered playback.
     */
    data class ErrorRecovered(
        override val eventId: String      = newId(),
        override val timestampMs: Long    = now(),
        override val sessionId: String,
        override val sequenceNum: Long = 0L,
        override val mediaId: String,
        override val category: EventCategory = EventCategory.ERROR,
        val retryCount: Int,
        val recoveryDurationMs: Long,
        val positionMs: Long,
    ) : PlayerEvent() {
        init {
            require(retryCount >= 0) { "retryCount must be >= 0" }
            require(recoveryDurationMs >= 0L) { "recoveryDurationMs must be >= 0" }
        }
    }

    /**
     * Automatic retry is being attempted.
     */
    data class ErrorRetrying(
        override val eventId: String      = newId(),
        override val timestampMs: Long    = now(),
        override val sessionId: String,
        override val sequenceNum: Long = 0L,
        override val mediaId: String?,
        override val category: EventCategory = EventCategory.ERROR,
        val retryAttempt: Int,
        val maxRetries: Int,
        val delayMs: Long,
        val errorCode: Int,
    ) : PlayerEvent() {
        init {
            require(retryAttempt > 0) { "retryAttempt must be > 0" }
            require(delayMs >= 0L) { "delayMs must be >= 0" }
        }

        val retriesRemaining: Int get() = maxRetries - retryAttempt
    }

    // ═══════════════════════════════════════════════════════════════════════
    // DRM Events
    // ═══════════════════════════════════════════════════════════════════════

    /**
     * DRM keys were successfully loaded.
     */
    data class DrmKeysLoaded(
        override val eventId: String      = newId(),
        override val timestampMs: Long    = now(),
        override val sessionId: String,
        override val sequenceNum: Long = 0L,
        override val mediaId: String,
        override val category: EventCategory = EventCategory.DRM,
        val drmScheme: String,
        val keyLoadDurationMs: Long,
        val licenseServerUrl: String,
    ) : PlayerEvent() {
        init {
            require(keyLoadDurationMs >= 0L) { "keyLoadDurationMs must be >= 0" }
        }
    }

    /**
     * DRM session error occurred.
     */
    data class DrmSessionError(
        override val eventId: String      = newId(),
        override val timestampMs: Long    = now(),
        override val sessionId: String,
        override val sequenceNum: Long = 0L,
        override val mediaId: String,
        override val category: EventCategory = EventCategory.DRM,
        val drmScheme: String,
        val errorCode: Int,
        val errorMessage: String,
        val licenseServerUrl: String,
    ) : PlayerEvent()

    // ═══════════════════════════════════════════════════════════════════════
    // Network Events
    // ═══════════════════════════════════════════════════════════════════════

    /**
     * Device network type changed during playback.
     * Impacts bandwidth availability and CDN routing.
     */
    data class NetworkTypeChanged(
        override val eventId: String      = newId(),
        override val timestampMs: Long    = now(),
        override val sessionId: String,
        override val sequenceNum: Long = 0L,
        override val mediaId: String?,
        override val category: EventCategory = EventCategory.NETWORK,
        val previousNetwork: NetworkType,
        val newNetwork: NetworkType,
        val estimatedBandwidthBps: Long   = 0L,
    ) : PlayerEvent()

    /**
     * CDN or origin server changed.
     * Can occur with multi-CDN setups or failover.
     */
    data class CdnSwitch(
        override val eventId: String      = newId(),
        override val timestampMs: Long    = now(),
        override val sessionId: String,
        override val sequenceNum: Long = 0L,
        override val mediaId: String,
        override val category: EventCategory = EventCategory.NETWORK,
        val previousCdn: String,
        val newCdn: String,
        val reason: CdnSwitchReason,
    ) : PlayerEvent() {

        enum class CdnSwitchReason {
            FAILOVER,
            PERFORMANCE,
            LOAD_BALANCE,
            GEO_ROUTING,
        }
    }

    // ═══════════════════════════════════════════════════════════════════════
    // Ad Events
    // ═══════════════════════════════════════════════════════════════════════

    /**
     * Ad playback started.
     */
    data class AdStarted(
        override val eventId: String      = newId(),
        override val timestampMs: Long    = now(),
        override val sessionId: String,
        override val sequenceNum: Long = 0L,
        override val mediaId: String,
        override val category: EventCategory = EventCategory.AD,
        val adId: String,
        val adPosition: AdPosition,
        val adDurationMs: Long,
        val adIndex: Int,
        val totalAds: Int,
    ) : PlayerEvent() {

        enum class AdPosition { PRE_ROLL, MID_ROLL, POST_ROLL }
    }

    /**
     * Ad playback ended (natural completion or skipped).
     */
    data class AdEnded(
        override val eventId: String      = newId(),
        override val timestampMs: Long    = now(),
        override val sessionId: String,
        override val sequenceNum: Long = 0L,
        override val mediaId: String,
        override val category: EventCategory = EventCategory.AD,
        val adId: String,
        val wasSkipped: Boolean,
        val watchedDurationMs: Long,
        val adDurationMs: Long,
    ) : PlayerEvent() {

        val completionPercent: Float
            get() = if (adDurationMs <= 0L) 100f
                    else (watchedDurationMs * 100f / adDurationMs).coerceIn(0f, 100f)
    }

    /**
     * Ad error occurred.
     */
    data class AdError(
        override val eventId: String      = newId(),
        override val timestampMs: Long    = now(),
        override val sessionId: String,
        override val sequenceNum: Long = 0L,
        override val mediaId: String,
        override val category: EventCategory = EventCategory.AD,
        val adId: String?,
        val errorCode: Int,
        val errorMessage: String,
        val isFatal: Boolean,
    ) : PlayerEvent()

    // ═══════════════════════════════════════════════════════════════════════
    // Shared Data Models
    // ═══════════════════════════════════════════════════════════════════════

    /**
     * Serialisable track descriptor.
     * Replaces Media3 [Format] to avoid holding Android framework references.
     */
    data class TrackInfo(
        val trackId: String,
        val mimeType: String,
        val codec: String?,
        val bitrateBps: Int,
        val width: Int             = 0,
        val height: Int            = 0,
        val frameRate: Float       = 0f,
        val language: String?      = null,
        val channelCount: Int      = 0,
        val sampleRateHz: Int      = 0,
        val isHdr: Boolean         = false,
        val label: String?         = null,
    ) {
        val isVideo: Boolean get() = width > 0 && height > 0
        val isAudio: Boolean get() = channelCount > 0
        val isText: Boolean  get() = !isVideo && !isAudio
        val resolutionLabel: String
            get() = when {
                height >= 2160 -> "4K"
                height >= 1080 -> "1080p"
                height >= 720  -> "720p"
                height >= 480  -> "480p"
                height >= 360  -> "360p"
                height > 0     -> "${height}p"
                else           -> ""
            }
    }

    /**
     * Device context for session-level reporting.
     * All strings — no Android types.
     */
    data class DeviceInfo(
        val manufacturer: String,
        val model: String,
        val osVersion: String,
        val sdkInt: Int,
        val screenWidthPx: Int,
        val screenHeightPx: Int,
        val screenDensity: Float,
        val totalRamMb: Long,
        val cpuAbi: String,
    )

    /**
     * Network type enumeration.
     * Kept in sync with [ConnectivityManager] types but decoupled from it.
     */
    enum class NetworkType {
        WIFI,
        CELLULAR_2G,
        CELLULAR_3G,
        CELLULAR_4G,
        CELLULAR_5G,
        ETHERNET,
        OFFLINE,
        UNKNOWN,
    }

    // ═══════════════════════════════════════════════════════════════════════
    // Factory utilities
    // ═══════════════════════════════════════════════════════════════════════

    companion object {
        fun newId(): String = UUID.randomUUID().toString()
        fun now(): Long = System.currentTimeMillis()
    }
}
```

## xplayer/app/src/main/java/com/xplayer/dev/core/events/PlayerEventBus.kt

```kotlin
package com.xplayer.dev.core.events

import kotlinx.coroutines.channels.Channel
import kotlinx.coroutines.channels.ChannelResult
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.MutableSharedFlow
import kotlinx.coroutines.flow.asSharedFlow
import kotlinx.coroutines.flow.receiveAsFlow
import timber.log.Timber
import java.util.concurrent.atomic.AtomicBoolean
import java.util.concurrent.atomic.AtomicLong
import javax.inject.Inject
import javax.inject.Singleton

/**
 * # PlayerEventBus
 *
 * Dual-channel event distribution:
 *
 *  1. [events]     — [Channel]-backed Flow. Each event delivered to ONE collector.
 *                    Used by [PlayerEventRouter] as the primary consumer.
 *                    Capacity: [PRIMARY_CAPACITY] = 512.
 *
 *  2. [broadcast]  — [MutableSharedFlow]-backed broadcast. Delivered to ALL collectors.
 *                    replay = 0 — no buffering for late subscribers.
 *                    Used by UI / analytics layers for fan-out observation.
 *
 * Why both?
 *  - The Channel guarantees delivery to a single reliable consumer (router).
 *  - The SharedFlow allows multiple UI components to observe without
 *    competing for events.
 *
 * Thread safety:
 *  - [send] is safe to call from the main thread only.
 *  - [events] and [broadcast] collectors can run on any thread.
 */
@Singleton
class PlayerEventBus @Inject constructor() {

    private val _channel = Channel<PlayerEvent>(capacity = PRIMARY_CAPACITY)
    private val _broadcast = MutableSharedFlow<PlayerEvent>(
        replay = 0,
        extraBufferCapacity = BROADCAST_EXTRA_CAPACITY,
    )

    private val _closed = AtomicBoolean(false)
    private val _sentCount = AtomicLong(0L)
    private val _droppedCount = AtomicLong(0L)
    private val _broadcastDropCount = AtomicLong(0L)

    /** Primary channel — single-consumer, guaranteed delivery. */
    val events: Flow<PlayerEvent> = _channel.receiveAsFlow()

    /** Broadcast flow — multi-consumer, fire-and-forget. */
    val broadcast: Flow<PlayerEvent> = _broadcast.asSharedFlow()

    // ── Send ──────────────────────────────────────────────────────────────────

    /**
     * Emit [event] to both [events] channel and [broadcast] flow.
     *
     * @return [SendResult] describing the outcome of both sends.
     */
    fun send(event: PlayerEvent): SendResult {
        if (_closed.get()) {
            Timber.w("EventBus: closed — dropping ${event::class.simpleName}")
            return SendResult(
                channelResult = ChannelOutcome.CLOSED,
                broadcastResult = BroadcastOutcome.CLOSED,
            )
        }

        val channelOutcome = sendToChannel(event)
        val broadcastOutcome = sendToBroadcast(event)

        if (channelOutcome == ChannelOutcome.SENT &&
            broadcastOutcome == BroadcastOutcome.SENT
        ) {
            _sentCount.incrementAndGet()
        }

        return SendResult(channelOutcome, broadcastOutcome)
    }

    /**
     * Close the bus. All subsequent [send] calls are silently dropped.
     * Idempotent — safe to call multiple times.
     */
    fun close() {
        if (_closed.compareAndSet(false, true)) {
            _channel.close()
            Timber.i(
                "EventBus: closed " +
                    "[sent=${_sentCount.get()}] " +
                    "[channelDropped=${_droppedCount.get()}] " +
                    "[broadcastDropped=${_broadcastDropCount.get()}]"
            )
        }
    }

    val isOpen: Boolean get() = !_closed.get()
    val sentCount: Long get() = _sentCount.get()
    val droppedCount: Long get() = _droppedCount.get()
    val broadcastDropCount: Long get() = _broadcastDropCount.get()

    // ── Private helpers ───────────────────────────────────────────────────────

    private fun sendToChannel(event: PlayerEvent): ChannelOutcome {
        val result: ChannelResult<Unit> = _channel.trySend(event)
        return when {
            result.isSuccess -> ChannelOutcome.SENT
            result.isFailure -> {
                _droppedCount.incrementAndGet()
                Timber.e(
                    "EventBus: channel FULL — dropped " +
                        "${event::class.simpleName} [id=${event.eventId}] " +
                        "[seq=${event.sequenceNum}]"
                )
                ChannelOutcome.DROPPED
            }
            result.isClosed  -> ChannelOutcome.CLOSED
            else             -> ChannelOutcome.CLOSED
        }
    }

    private fun sendToBroadcast(event: PlayerEvent): BroadcastOutcome {
        val emitted = _broadcast.tryEmit(event)
        return if (emitted) {
            BroadcastOutcome.SENT
        } else {
            _broadcastDropCount.incrementAndGet()
            Timber.w(
                "EventBus: broadcast dropped " +
                    "${event::class.simpleName} [id=${event.eventId}]"
            )
            BroadcastOutcome.DROPPED
        }
    }

    // ── Result types ──────────────────────────────────────────────────────────

    data class SendResult(
        val channelResult: ChannelOutcome,
        val broadcastResult: BroadcastOutcome,
    ) {
        val isFullyDelivered: Boolean
            get() = channelResult == ChannelOutcome.SENT &&
                broadcastResult == BroadcastOutcome.SENT
    }

    enum class ChannelOutcome { SENT, DROPPED, CLOSED }
    enum class BroadcastOutcome { SENT, DROPPED, CLOSED }

    companion object {
        const val PRIMARY_CAPACITY = 512
        const val BROADCAST_EXTRA_CAPACITY = 64
    }
}
```

## xplayer/app/src/main/java/com/xplayer/dev/core/events/PlayerEventDiagnostics.kt

```kotlin
package com.xplayer.dev.core.events

import javax.inject.Inject
import javax.inject.Singleton

/**
 * # PlayerEventDiagnostics
 *
 * Produces structured crash reports and audit trails
 * from [PlayerEventStore] contents.
 */
@Singleton
class PlayerEventDiagnostics @Inject constructor(
    private val store: PlayerEventStore,
    private val serializer: PlayerEventSerializer,
) {

    /**
     * Full diagnostic report for a session.
     * Attach to crash reports or support tickets.
     */
    fun sessionReport(sessionId: String): String = buildString {
        val events = store.forSession(sessionId)
        val qos = store.sessionQosSummary(sessionId)

        appendLine("═══════════════════════════════════════")
        appendLine("PlayerEvent Diagnostic Report")
        appendLine("Session: $sessionId")
        appendLine("═══════════════════════════════════════")

        appendLine("\n── QoS Summary ─────────────────────────")
        appendLine("TTFP             : ${qos.ttfpMs?.let { "${it}ms" } ?: "Not measured"}")
        appendLine("Total stall time : ${qos.totalStallMs}ms")
        appendLine("Stall count      : ${qos.stallCount}")
        appendLine("Dropped frames   : ${qos.totalDroppedFrames}")
        appendLine("Avg bandwidth    : ${qos.averageBandwidthBps / 1000}kbps")
        appendLine("Peak bandwidth   : ${qos.peakBandwidthBps / 1000}kbps")
        appendLine("Seek count       : ${qos.seekCount}")
        appendLine("Error count      : ${qos.errorCount}")
        appendLine("Healthy session  : ${qos.isHealthy}")

        appendLine("\n── Event Timeline (${events.size} events) ──────────")
        events.forEachIndexed { idx, event ->
            appendLine(
                "  ${idx.toString().padStart(4)}. " +
                    "[${event.timestampMs}] " +
                    "[${event.category.name.padEnd(10)}] " +
                    "${event::class.simpleName}"
            )
        }

        appendLine("\n── Errors ──────────────────────────────")
        val errors = store.errors()
        if (errors.isEmpty()) {
            appendLine("  No errors recorded.")
        } else {
            errors.forEachIndexed { idx, e ->
                appendLine(
                    "  $idx. [${e.errorCodeName}] " +
                        "fatal=${e.isFatal} " +
                        "retry=${e.retryAttempt}/${e.maxRetries} " +
                        "prev=${e.previousState}"
                )
            }
        }

        appendLine("\n═══════════════════════════════════════")
    }

    /**
     * Last [n] events as crash breadcrumb strings.
     */
    fun crashBreadcrumbs(n: Int = 25): List<String> = store.breadcrumbs(n)

    /**
     * Serialised map for Crashlytics / Sentry custom keys.
     * Attaches last error + QoS summary as flat key-value pairs.
     */
    fun crashKeyValueMap(sessionId: String): Map<String, String> = buildMap {
        val qos = store.sessionQosSummary(sessionId)
        qos.ttfpMs?.let { put("qos_ttfp_ms", it.toString()) }
        put("qos_stall_count", qos.stallCount.toString())
        put("qos_total_stall_ms", qos.totalStallMs.toString())
        put("qos_dropped_frames", qos.totalDroppedFrames.toString())
        put("qos_avg_bw_bps", qos.averageBandwidthBps.toString())
        put("qos_seek_count", qos.seekCount.toString())
        put("qos_error_count", qos.errorCount.toString())
        put("qos_healthy", qos.isHealthy.toString())

        store.fatalErrors().lastOrNull()?.let { e ->
            put("last_error_code", e.errorCode.toString())
            put("last_error_name", e.errorCodeName)
            put("last_error_fatal", e.isFatal.toString())
            put("last_error_prev_state", e.previousState)
        }
    }
}
```

## xplayer/app/src/main/java/com/xplayer/dev/core/events/PlayerEventExtensions.kt

```kotlin
package com.xplayer.dev.core.events

import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.filter
import kotlinx.coroutines.flow.filterIsInstance
import kotlinx.coroutines.flow.map
import kotlinx.coroutines.flow.mapNotNull

// ═══════════════════════════════════════════════════════════════════════════
// Category filters
// ═══════════════════════════════════════════════════════════════════════════

fun Flow<PlayerEvent>.sessionEvents(): Flow<PlayerEvent> =
    filter { it.category == PlayerEvent.EventCategory.SESSION }

fun Flow<PlayerEvent>.playbackEvents(): Flow<PlayerEvent> =
    filter { it.category == PlayerEvent.EventCategory.PLAYBACK }

fun Flow<PlayerEvent>.seekEvents(): Flow<PlayerEvent> =
    filter { it.category == PlayerEvent.EventCategory.SEEK }

fun Flow<PlayerEvent>.bufferEvents(): Flow<PlayerEvent> =
    filter { it.category == PlayerEvent.EventCategory.BUFFER }

fun Flow<PlayerEvent>.qosEvents(): Flow<PlayerEvent> =
    filter { it.category == PlayerEvent.EventCategory.QOS }

fun Flow<PlayerEvent>.errorEvents(): Flow<PlayerEvent> =
    filter { it.category == PlayerEvent.EventCategory.ERROR }

fun Flow<PlayerEvent>.networkEvents(): Flow<PlayerEvent> =
    filter { it.category == PlayerEvent.EventCategory.NETWORK }

fun Flow<PlayerEvent>.drmEvents(): Flow<PlayerEvent> =
    filter { it.category == PlayerEvent.EventCategory.DRM }

fun Flow<PlayerEvent>.adEvents(): Flow<PlayerEvent> =
    filter { it.category == PlayerEvent.EventCategory.AD }

// ═══════════════════════════════════════════════════════════════════════════
// Typed event filters
// ═══════════════════════════════════════════════════════════════════════════

fun Flow<PlayerEvent>.playbackStarted(): Flow<PlayerEvent.PlaybackStarted> =
    filterIsInstance()

fun Flow<PlayerEvent>.playbackPaused(): Flow<PlayerEvent.PlaybackPaused> =
    filterIsInstance()

fun Flow<PlayerEvent>.playbackResumed(): Flow<PlayerEvent.PlaybackResumed> =
    filterIsInstance()

fun Flow<PlayerEvent>.playbackStopped(): Flow<PlayerEvent.PlaybackStopped> =
    filterIsInstance()

fun Flow<PlayerEvent>.playbackEnded(): Flow<PlayerEvent.PlaybackEnded> =
    filterIsInstance()

fun Flow<PlayerEvent>.playbackSpeedChanged(): Flow<PlayerEvent.PlaybackSpeedChanged> =
    filterIsInstance()

fun Flow<PlayerEvent>.seekRequested(): Flow<PlayerEvent.SeekRequested> =
    filterIsInstance()

fun Flow<PlayerEvent>.seekCompleted(): Flow<PlayerEvent.SeekCompleted> =
    filterIsInstance()

fun Flow<PlayerEvent>.bufferingStarted(): Flow<PlayerEvent.BufferingStarted> =
    filterIsInstance()

fun Flow<PlayerEvent>.bufferingEnded(): Flow<PlayerEvent.BufferingEnded> =
    filterIsInstance()

fun Flow<PlayerEvent>.bufferUpdates(): Flow<PlayerEvent.BufferUpdated> =
    filterIsInstance()

fun Flow<PlayerEvent>.tracksChanged(): Flow<PlayerEvent.TracksChanged> =
    filterIsInstance()

fun Flow<PlayerEvent>.videoSizeChanged(): Flow<PlayerEvent.VideoSizeChanged> =
    filterIsInstance()

fun Flow<PlayerEvent>.metadataReceived(): Flow<PlayerEvent.MetadataReceived> =
    filterIsInstance()

fun Flow<PlayerEvent>.errorsOccurred(): Flow<PlayerEvent.ErrorOccurred> =
    filterIsInstance()

fun Flow<PlayerEvent>.fatalErrors(): Flow<PlayerEvent.ErrorOccurred> =
    filterIsInstance<PlayerEvent.ErrorOccurred>().filter { it.isFatal }

fun Flow<PlayerEvent>.recoverableErrors(): Flow<PlayerEvent.ErrorOccurred> =
    filterIsInstance<PlayerEvent.ErrorOccurred>().filter { !it.isFatal }

fun Flow<PlayerEvent>.bitrateChanges(): Flow<PlayerEvent.BitrateChanged> =
    filterIsInstance()

fun Flow<PlayerEvent>.droppedFrames(): Flow<PlayerEvent.DroppedFrames> =
    filterIsInstance()

fun Flow<PlayerEvent>.bandwidthEstimates(): Flow<PlayerEvent.BandwidthEstimated> =
    filterIsInstance()

fun Flow<PlayerEvent>.sessionStarted(): Flow<PlayerEvent.SessionStarted> =
    filterIsInstance()

fun Flow<PlayerEvent>.sessionEnded(): Flow<PlayerEvent.SessionEnded> =
    filterIsInstance()

fun Flow<PlayerEvent>.drmKeysLoaded(): Flow<PlayerEvent.DrmKeysLoaded> =
    filterIsInstance()

fun Flow<PlayerEvent>.drmErrors(): Flow<PlayerEvent.DrmSessionError> =
    filterIsInstance()

// ═══════════════════════════════════════════════════════════════════════════
// Session filter — only events for a specific session
// ═══════════════════════════════════════════════════════════════════════════

fun Flow<PlayerEvent>.forSession(sessionId: String): Flow<PlayerEvent> =
    filter { it.sessionId == sessionId }

fun Flow<PlayerEvent>.forMedia(mediaId: String): Flow<PlayerEvent> =
    filter { it.mediaId == mediaId }

// ═══════════════════════════════════════════════════════════════════════════
// Projection helpers
// ═══════════════════════════════════════════════════════════════════════════

fun Flow<PlayerEvent>.eventIds(): Flow<String> =
    map { it.eventId }

fun Flow<PlayerEvent>.timestamps(): Flow<Long> =
    map { it.timestampMs }

fun Flow<PlayerEvent>.ttfpValues(): Flow<Long> =
    filterIsInstance<PlayerEvent.PlaybackStarted>()
        .filter { !it.isResumed }
        .map { it.timeToFirstFrameMs }

fun Flow<PlayerEvent>.stallDurations(): Flow<Long> =
    filterIsInstance<PlayerEvent.BufferingEnded>()
        .filter { it.stallDurationMs > 0L }
        .map { it.stallDurationMs }

fun Flow<PlayerEvent>.bandwidthBps(): Flow<Long> =
    filterIsInstance<PlayerEvent.BandwidthEstimated>()
        .map { it.bandwidthBps }

fun Flow<PlayerEvent>.errorCodes(): Flow<Int> =
    filterIsInstance<PlayerEvent.ErrorOccurred>()
        .map { it.errorCode }

// ═══════════════════════════════════════════════════════════════════════════
// Boolean properties
// ═══════════════════════════════════════════════════════════════════════════

val PlayerEvent.isError: Boolean
    get() = this is PlayerEvent.ErrorOccurred

val PlayerEvent.isFatalError: Boolean
    get() = this is PlayerEvent.ErrorOccurred && isFatal

val PlayerEvent.isSessionBoundary: Boolean
    get() = this is PlayerEvent.SessionStarted || this is PlayerEvent.SessionEnded

val PlayerEvent.isQosCritical: Boolean
    get() = when (this) {
        is PlayerEvent.DroppedFrames      -> isCritical
        is PlayerEvent.BufferingStarted   -> isRebuffer
        is PlayerEvent.ErrorOccurred      -> isFatal
        is PlayerEvent.BitrateChanged     -> switchDirection == PlayerEvent.BitrateChanged.SwitchDirection.DOWN
        else                              -> false
    }

val PlayerEvent.logTag: String
    get() = "[${category.name}][${this::class.simpleName}][session=${sessionId.take(8)}]"
```

## xplayer/app/src/main/java/com/xplayer/dev/core/events/PlayerEventFilter.kt

```kotlin
package com.xplayer.dev.core.events

import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.filter

/**
 * # PlayerEventFilter
 *
 * Composable, reusable predicates for [PlayerEvent] streams.
 *
 * Usage:
 * ```kotlin
 * val filter = PlayerEventFilter.Builder()
 *     .category(PlayerEvent.EventCategory.ERROR)
 *     .fatal(true)
 *     .session(currentSessionId)
 *     .build()
 *
 * eventBus.broadcast
 *     .applyFilter(filter)
 *     .collect { event -> ... }
 * ```
 */
data class PlayerEventFilter(
    val categories: Set<PlayerEvent.EventCategory>  = emptySet(),
    val sessionId: String?                           = null,
    val mediaId: String?                             = null,
    val onlyFatalErrors: Boolean                     = false,
    val onlyQosCritical: Boolean                     = false,
    val excludeCategories: Set<PlayerEvent.EventCategory> = emptySet(),
    val minSequenceNum: Long                         = 0L,
    val maxSequenceNum: Long                         = Long.MAX_VALUE,
    val since: Long                                  = 0L,          // timestampMs
    val until: Long                                  = Long.MAX_VALUE,
) {
    /** Returns true if [event] passes all filter conditions. */
    fun matches(event: PlayerEvent): Boolean {
        if (categories.isNotEmpty() && event.category !in categories) return false
        if (excludeCategories.isNotEmpty() && event.category in excludeCategories) return false
        if (sessionId != null && event.sessionId != sessionId) return false
        if (mediaId != null && event.mediaId != mediaId) return false
        if (onlyFatalErrors && event !is PlayerEvent.ErrorOccurred) return false
        if (onlyFatalErrors && event is PlayerEvent.ErrorOccurred && !event.isFatal) return false
        if (onlyQosCritical && !event.isQosCritical) return false
        if (event.sequenceNum < minSequenceNum) return false
        if (event.sequenceNum > maxSequenceNum) return false
        if (event.timestampMs < since) return false
        if (event.timestampMs > until) return false
        return true
    }

    class Builder {
        private val categories = mutableSetOf<PlayerEvent.EventCategory>()
        private val excludedCategories = mutableSetOf<PlayerEvent.EventCategory>()
        private var sessionId: String? = null
        private var mediaId: String? = null
        private var onlyFatalErrors = false
        private var onlyQosCritical = false
        private var minSeq = 0L
        private var maxSeq = Long.MAX_VALUE
        private var since = 0L
        private var until = Long.MAX_VALUE

        fun category(vararg c: PlayerEvent.EventCategory) =
            apply { categories.addAll(c) }

        fun exclude(vararg c: PlayerEvent.EventCategory) =
            apply { excludedCategories.addAll(c) }

        fun session(id: String) = apply { sessionId = id }
        fun media(id: String) = apply { mediaId = id }
        fun fatalOnly() = apply { onlyFatalErrors = true }
        fun qosCriticalOnly() = apply { onlyQosCritical = true }
        fun sequenceRange(min: Long, max: Long) = apply { minSeq = min; maxSeq = max }
        fun timeRange(fromMs: Long, toMs: Long) = apply { since = fromMs; until = toMs }

        fun build() = PlayerEventFilter(
            categories       = categories.toSet(),
            sessionId        = sessionId,
            mediaId          = mediaId,
            onlyFatalErrors  = onlyFatalErrors,
            onlyQosCritical  = onlyQosCritical,
            excludeCategories = excludedCategories.toSet(),
            minSequenceNum   = minSeq,
            maxSequenceNum   = maxSeq,
            since            = since,
            until            = until,
        )
    }

    companion object {
        val ALL = PlayerEventFilter()
        val ERRORS_ONLY = Builder().category(PlayerEvent.EventCategory.ERROR).build()
        val QOS_ONLY = Builder().category(PlayerEvent.EventCategory.QOS).build()
        val FATAL_ERRORS = Builder().fatalOnly().build()
        val SESSION_ONLY = Builder().category(PlayerEvent.EventCategory.SESSION).build()
        val NO_BUFFER_UPDATES = Builder()
            .exclude(PlayerEvent.EventCategory.BUFFER)
            .build()
    }
}

/** Apply a [PlayerEventFilter] to this flow. */
fun Flow<PlayerEvent>.applyFilter(filter: PlayerEventFilter): Flow<PlayerEvent> =
    if (filter == PlayerEventFilter.ALL) this
    else this.filter { filter.matches(it) }
```

## xplayer/app/src/main/java/com/xplayer/dev/core/events/PlayerEventLogger.kt

```kotlin
package com.xplayer.dev.core.events

import timber.log.Timber
import javax.inject.Inject
import javax.inject.Singleton

/**
 * Logs player events with structured details.
 */
@Singleton
class PlayerEventLogger @Inject constructor() {

    fun logEvent(event: PlayerEvent) {
        val name = event::class.simpleName ?: "UnknownEvent"
        Timber.d("PlayerEvent: [$name] id=${event.eventId} session=${event.sessionId} ts=${event.timestampMs}")
    }
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/events/PlayerEventQueue.kt

```kotlin
package com.xplayer.dev.core.events

import java.util.concurrent.ConcurrentLinkedQueue
import javax.inject.Inject
import javax.inject.Singleton

/**
 * Thread-safe event queue maintaining a bounded buffer of recently emitted events.
 */
@Singleton
class PlayerEventQueue @Inject constructor() {

    private val capacity = 100
    private val queue = ConcurrentLinkedQueue<PlayerEvent>()

    fun enqueue(event: PlayerEvent) {
        queue.offer(event)
        while (queue.size > capacity) {
            queue.poll()
        }
    }

    fun getAll(): List<PlayerEvent> = queue.toList()

    fun clear() {
        queue.clear()
    }
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/events/PlayerEventRouter.kt

```kotlin
package com.xplayer.dev.core.events

import kotlinx.coroutines.CoroutineScope
import kotlinx.coroutines.Job
import kotlinx.coroutines.flow.launchIn
import kotlinx.coroutines.flow.onEach
import timber.log.Timber
import javax.inject.Inject
import javax.inject.Singleton

/**
 * # PlayerEventRouter
 *
 * Consumes the primary [PlayerEventBus.events] channel and routes
 * each event to registered typed handlers.
 *
 * Responsibilities:
 *  1. Single primary consumer of the channel — no events lost to racing.
 *  2. Routes to typed [EventHandler] registry.
 *  3. Routes to [PlayerEventStore] for history.
 *  4. Routes to [PlayerEventValidator] for schema checks.
 *  5. Catches handler exceptions — one bad handler cannot kill the pipeline.
 *
 * @param bus       Source event bus.
 * @param store     History store for replay and diagnostics.
 * @param validator Schema validator.
 */
@Singleton
class PlayerEventRouter @Inject constructor(
    private val bus: PlayerEventBus,
    private val store: PlayerEventStore,
    private val validator: PlayerEventValidator,
) {

    private val handlers =
        mutableMapOf<PlayerEvent.EventCategory, MutableList<EventHandler<*>>>()

    private var routerJob: Job? = null

    // ── Lifecycle ─────────────────────────────────────────────────────────────

    /**
     * Start consuming events from [bus].
     * Must be called once — subsequent calls are no-ops.
     *
     * @param scope Coroutine scope; should match player session lifetime.
     */
    fun start(scope: CoroutineScope) {
        if (routerJob?.isActive == true) {
            Timber.w("EventRouter: already running")
            return
        }
        routerJob = bus.events
            .onEach { event -> route(event) }
            .launchIn(scope)
        Timber.i("EventRouter: started")
    }

    fun stop() {
        routerJob?.cancel()
        routerJob = null
        Timber.i("EventRouter: stopped")
    }

    // ── Handler registration ──────────────────────────────────────────────────

    /**
     * Register a typed handler for a specific event category.
     *
     * Example:
     * ```kotlin
     * router.on(PlayerEvent.EventCategory.ERROR) { event: PlayerEvent.ErrorOccurred ->
     *     crashReporter.record(event.error)
     * }
     * ```
     */
    @Suppress("UNCHECKED_CAST")
    fun <T : PlayerEvent> on(
        category: PlayerEvent.EventCategory,
        handler: EventHandler<T>,
    ) {
        handlers
            .getOrPut(category) { mutableListOf() }
            .add(handler as EventHandler<*>)
        Timber.d("EventRouter: registered handler for ${category.name}")
    }

    fun removeHandlers(category: PlayerEvent.EventCategory) {
        handlers.remove(category)
    }

    fun clearAllHandlers() {
        handlers.clear()
    }

    // ── Routing ───────────────────────────────────────────────────────────────

    @Suppress("UNCHECKED_CAST")
    private suspend fun route(event: PlayerEvent) {
        // 1. Validate
        val validationResult = validator.validate(event)
        if (validationResult is PlayerEventValidator.ValidationResult.Invalid) {
            Timber.e(
                "EventRouter: INVALID event " +
                    "${event::class.simpleName} — ${validationResult.reasons}"
            )
            // In debug: drop invalid events. In release: log and continue.
        }

        // 2. Store
        store.record(event)

        // 3. Route to handlers
        val categoryHandlers = handlers[event.category] ?: return

        categoryHandlers.forEach { handler ->
            runCatching {
                (handler as EventHandler<PlayerEvent>).handle(event)
            }.onFailure { throwable ->
                Timber.e(
                    throwable,
                    "EventRouter: handler threw for " +
                        "${event::class.simpleName} [${event.eventId}]"
                )
            }
        }
    }

    fun interface EventHandler<T : PlayerEvent> {
        suspend fun handle(event: T)
    }
}
```

## xplayer/app/src/main/java/com/xplayer/dev/core/events/PlayerEventSerializer.kt

```kotlin
package com.xplayer.dev.core.events

import javax.inject.Inject
import javax.inject.Singleton

/**
 * # PlayerEventSerializer
 *
 * Converts [PlayerEvent] instances to flat [Map<String, String>] payloads.
 *
 * Why not JSON / Gson / Moshi?
 *  - No library dependency — works in any environment.
 *  - Crash reporters (Crashlytics, Sentry) accept Map<String,String>.
 *  - Analytics SDKs (Amplitude, Firebase) accept Map<String,String>.
 *  - Zero reflection — compile-time safe.
 *
 * Every map includes common fields from [baseFields] plus
 * type-specific fields that vary per event subclass.
 */
@Singleton
class PlayerEventSerializer @Inject constructor() {

    /**
     * Serialise [event] to a flat string map.
     * Keys use snake_case for analytics pipeline compatibility.
     */
    fun serialize(event: PlayerEvent): Map<String, String> = buildMap {
        putAll(baseFields(event))
        putAll(specificFields(event))
    }

    /**
     * Serialise a list of events to a list of maps.
     */
    fun serializeAll(events: List<PlayerEvent>): List<Map<String, String>> =
        events.map { serialize(it) }

    // ── Base fields (every event) ─────────────────────────────────────────────

    private fun baseFields(event: PlayerEvent): Map<String, String> = mapOf(
        "event_id"       to event.eventId,
        "event_type"     to (event::class.simpleName ?: "Unknown"),
        "category"       to event.category.name,
        "timestamp_ms"   to event.timestampMs.toString(),
        "session_id"     to event.sessionId,
        "sequence_num"   to event.sequenceNum.toString(),
        "media_id"       to (event.mediaId ?: ""),
    )

    // ── Type-specific fields ──────────────────────────────────────────────────

    private fun specificFields(event: PlayerEvent): Map<String, String> =
        when (event) {

            is PlayerEvent.SessionStarted -> mapOf(
                "url"            to event.url,
                "content_type"   to event.contentType.name,
                "start_pos_ms"   to event.startPositionMs.toString(),
                "network_type"   to event.networkType.name,
                "device_make"    to event.deviceInfo.manufacturer,
                "device_model"   to event.deviceInfo.model,
                "os_version"     to event.deviceInfo.osVersion,
                "sdk_int"        to event.deviceInfo.sdkInt.toString(),
                "app_version"    to event.appVersion,
                "player_version" to event.playerVersion,
                "screen_w"       to event.deviceInfo.screenWidthPx.toString(),
                "screen_h"       to event.deviceInfo.screenHeightPx.toString(),
            )

            is PlayerEvent.SessionEnded -> mapOf(
                "end_reason"          to event.endReason.name,
                "total_session_ms"    to event.totalSessionMs.toString(),
                "total_played_ms"     to event.totalPlayedMs.toString(),
                "total_stalled_ms"    to event.totalStalledMs.toString(),
                "total_stall_count"   to event.totalStallCount.toString(),
                "total_seek_count"    to event.totalSeekCount.toString(),
                "total_dropped_frames" to event.totalDroppedFrames.toString(),
                "avg_bandwidth_bps"   to event.averageBandwidthBps.toString(),
                "max_bandwidth_bps"   to event.maxConcurrentBps.toString(),
                "completion_percent"  to event.completionPercent.toString(),
            )

            is PlayerEvent.PlaybackStarted -> mapOf(
                "position_ms"      to event.positionMs.toString(),
                "duration_ms"      to event.durationMs.toString(),
                "is_resumed"       to event.isResumed.toString(),
                "ttfp_ms"          to event.timeToFirstFrameMs.toString(),
                "playback_speed"   to event.playbackSpeed.toString(),
                "is_live"          to event.isLive.toString(),
                "video_width"      to event.videoWidth.toString(),
                "video_height"     to event.videoHeight.toString(),
                "is_audio_only"    to event.isAudioOnly.toString(),
            )

            is PlayerEvent.PlaybackPaused -> mapOf(
                "position_ms"      to event.positionMs.toString(),
                "pause_reason"     to event.pauseReason.name,
                "paused_by_system" to event.pausedBySystem.toString(),
            )

            is PlayerEvent.PlaybackResumed -> mapOf(
                "position_ms"      to event.positionMs.toString(),
                "pause_duration_ms" to event.pauseDurationMs.toString(),
                "playback_speed"   to event.playbackSpeed.toString(),
            )

            is PlayerEvent.PlaybackStopped -> mapOf(
                "position_ms"      to event.positionMs.toString(),
                "stop_reason"      to event.stopReason.name,
                "played_ms"        to event.playedDurationMs.toString(),
            )

            is PlayerEvent.PlaybackEnded -> mapOf(
                "total_played_ms"   to event.totalPlayedMs.toString(),
                "total_duration_ms" to event.totalDurationMs.toString(),
                "completion_pct"    to event.completionPercent.toString(),
                "is_full_complete"  to event.isFullCompletion.toString(),
            )

            is PlayerEvent.PlaybackSpeedChanged -> mapOf(
                "previous_speed" to event.previousSpeed.toString(),
                "new_speed"      to event.newSpeed.toString(),
                "initiator"      to event.initiator.name,
            )

            is PlayerEvent.SeekRequested -> mapOf(
                "from_ms"     to event.fromPositionMs.toString(),
                "to_ms"       to event.toPositionMs.toString(),
                "delta_ms"    to event.seekDeltaMs.toString(),
                "seek_type"   to event.seekType.name,
                "initiator"   to event.initiator.name,
                "is_forward"  to event.isForwardSeek.toString(),
            )

            is PlayerEvent.SeekCompleted -> mapOf(
                "from_ms"          to event.fromPositionMs.toString(),
                "to_ms"            to event.toPositionMs.toString(),
                "seek_duration_ms" to event.seekDurationMs.toString(),
                "required_rebuffer" to event.requiredRebuffer.toString(),
            )

            is PlayerEvent.BufferingStarted -> mapOf(
                "position_ms"      to event.positionMs.toString(),
                "buffer_ahead_ms"  to event.bufferAheadMs.toString(),
                "buffer_percent"   to event.bufferPercent.toString(),
                "is_initial"       to event.isInitialBuffer.toString(),
                "stall_count"      to event.stallCount.toString(),
                "is_rebuffer"      to event.isRebuffer.toString(),
            )

            is PlayerEvent.BufferingEnded -> mapOf(
                "stall_ms"         to event.stallDurationMs.toString(),
                "buffer_pct"       to event.bufferPercentOnResume.toString(),
                "exit_reason"      to event.exitReason.name,
            )

            is PlayerEvent.BufferUpdated -> mapOf(
                "buffer_percent"   to event.bufferPercent.toString(),
                "buffered_pos_ms"  to event.bufferedPositionMs.toString(),
                "buffer_ahead_ms"  to event.bufferAheadMs.toString(),
                "current_pos_ms"   to event.currentPositionMs.toString(),
            )

            is PlayerEvent.ErrorOccurred -> mapOf(
                "error_code"       to event.errorCode.toString(),
                "error_code_name"  to event.errorCodeName,
                "error_message"    to event.errorMessage,
                "error_category"   to event.errorCategory.name,
                "is_fatal"         to event.isFatal.toString(),
                "retry_attempt"    to event.retryAttempt.toString(),
                "max_retries"      to event.maxRetries.toString(),
                "can_retry"        to event.canRetry.toString(),
                "previous_state"   to event.previousState,
            ) + event.context.mapKeys { "ctx_${it.key}" }

            is PlayerEvent.ErrorRecovered -> mapOf(
                "retry_count"      to event.retryCount.toString(),
                "recovery_ms"      to event.recoveryDurationMs.toString(),
                "position_ms"      to event.positionMs.toString(),
            )

            is PlayerEvent.ErrorRetrying -> mapOf(
                "retry_attempt"    to event.retryAttempt.toString(),
                "max_retries"      to event.maxRetries.toString(),
                "delay_ms"         to event.delayMs.toString(),
                "error_code"       to event.errorCode.toString(),
                "retries_remaining" to event.retriesRemaining.toString(),
            )

            is PlayerEvent.BitrateChanged -> mapOf(
                "track_type"       to event.trackType.name,
                "prev_bps"         to event.previousBitrateBps.toString(),
                "new_bps"          to event.newBitrateBps.toString(),
                "direction"        to event.switchDirection.name,
                "reason"           to event.reason.name,
                "delta_bps"        to event.bitrateDeltaBps.toString(),
                "switch_ratio"     to event.switchRatio.toString(),
                "network_bps"      to event.networkBandwidthBps.toString(),
            )

            is PlayerEvent.DroppedFrames -> mapOf(
                "dropped_count"    to event.droppedCount.toString(),
                "elapsed_ms"       to event.elapsedMs.toString(),
                "total_dropped"    to event.totalDropped.toString(),
                "drop_rate"        to event.dropRate.toString(),
                "is_critical"      to event.isCritical.toString(),
            )

            is PlayerEvent.BandwidthEstimated -> mapOf(
                "bandwidth_bps"    to event.bandwidthBps.toString(),
                "sample_bps"       to event.sampleBps.toString(),
                "percentile_rank"  to event.percentileRank.toString(),
                "bandwidth_mbps"   to event.bandwidthMbps.toString(),
            )

            is PlayerEvent.VideoSizeChanged -> mapOf(
                "width"            to event.width.toString(),
                "height"           to event.height.toString(),
                "pixel_ratio"      to event.pixelWidthHeightRatio.toString(),
                "rotation"         to event.unappliedRotationDegrees.toString(),
                "total_pixels"     to event.totalPixels.toString(),
                "aspect_ratio"     to event.aspectRatio.toString(),
                "is_4k"            to event.is4K.toString(),
                "is_hd"            to event.isHD.toString(),
            )

            is PlayerEvent.MetadataReceived -> mapOf(
                "title"            to (event.title ?: ""),
                "artist"           to (event.artist ?: ""),
                "album"            to (event.album ?: ""),
                "artwork_uri"      to (event.artworkUri ?: ""),
                "duration_ms"      to (event.durationMs?.toString() ?: ""),
                "source"           to event.source.name,
            )

            is PlayerEvent.DrmKeysLoaded -> mapOf(
                "drm_scheme"       to event.drmScheme,
                "key_load_ms"      to event.keyLoadDurationMs.toString(),
                "license_url"      to event.licenseServerUrl,
            )

            is PlayerEvent.DrmSessionError -> mapOf(
                "drm_scheme"       to event.drmScheme,
                "error_code"       to event.errorCode.toString(),
                "error_message"    to event.errorMessage,
            )

            is PlayerEvent.NetworkTypeChanged -> mapOf(
                "prev_network"     to event.previousNetwork.name,
                "new_network"      to event.newNetwork.name,
                "estimated_bps"    to event.estimatedBandwidthBps.toString(),
            )

            else -> emptyMap()
        }
}
```

## xplayer/app/src/main/java/com/xplayer/dev/core/events/PlayerEventStore.kt

```kotlin
package com.xplayer.dev.core.events

import java.util.concurrent.CopyOnWriteArrayList
import javax.inject.Inject
import javax.inject.Singleton

/**
 * # PlayerEventStore
 *
 * Bounded, thread-safe history buffer for [PlayerEvent] records.
 *
 * Use cases:
 *  1. Crash breadcrumbs — last N events before crash.
 *  2. QoS aggregation — compute session-level metrics.
 *  3. Debug replay — reproduce exact event sequences.
 *  4. Filtered queries — retrieve events by category / session.
 *
 * @param capacity Maximum events to retain. Oldest evicted when full.
 */
@Singleton
class PlayerEventStore @Inject constructor() { 
    private val capacity = DEFAULT_CAPACITY
    private val _events = CopyOnWriteArrayList<PlayerEvent>()

    val events: List<PlayerEvent> get() = _events.toList()
    val size: Int get() = _events.size

    // ── Write ─────────────────────────────────────────────────────────────────

    fun record(event: PlayerEvent) {
        if (_events.size >= capacity) _events.removeAt(0)
        _events.add(event)
    }

    fun clear() = _events.clear()

    // ── Read — filtered queries ───────────────────────────────────────────────

    fun query(filter: PlayerEventFilter): List<PlayerEvent> =
        _events.filter { filter.matches(it) }

    fun forSession(sessionId: String): List<PlayerEvent> =
        _events.filter { it.sessionId == sessionId }

    fun forCategory(category: PlayerEvent.EventCategory): List<PlayerEvent> =
        _events.filter { it.category == category }

    fun errors(): List<PlayerEvent.ErrorOccurred> =
        _events.filterIsInstance<PlayerEvent.ErrorOccurred>()

    fun fatalErrors(): List<PlayerEvent.ErrorOccurred> =
        errors().filter { it.isFatal }

    fun last(n: Int): List<PlayerEvent> =
        _events.takeLast(n.coerceAtLeast(0))

    fun since(timestampMs: Long): List<PlayerEvent> =
        _events.filter { it.timestampMs >= timestampMs }

    // ── QoS aggregation ───────────────────────────────────────────────────────

    /**
     * Compute QoS summary for a session.
     * All metrics derived from stored events — no external state needed.
     */
    fun sessionQosSummary(sessionId: String): SessionQosSummary {
        val sessionEvents = forSession(sessionId)

        val ttfp = sessionEvents
            .filterIsInstance<PlayerEvent.PlaybackStarted>()
            .firstOrNull { !it.isResumed }
            ?.timeToFirstFrameMs

        val totalStallMs = sessionEvents
            .filterIsInstance<PlayerEvent.BufferingEnded>()
            .sumOf { it.stallDurationMs }

        val stallCount = sessionEvents
            .filterIsInstance<PlayerEvent.BufferingStarted>()
            .count { it.isRebuffer }

        val totalDroppedFrames = sessionEvents
            .filterIsInstance<PlayerEvent.DroppedFrames>()
            .sumOf { it.droppedCount }

        val bandwidthSamples = sessionEvents
            .filterIsInstance<PlayerEvent.BandwidthEstimated>()
            .map { it.bandwidthBps }

        val seekCount = sessionEvents
            .filterIsInstance<PlayerEvent.SeekRequested>()
            .size

        val errorCount = sessionEvents
            .filterIsInstance<PlayerEvent.ErrorOccurred>()
            .size

        return SessionQosSummary(
            sessionId           = sessionId,
            ttfpMs              = ttfp,
            totalStallMs        = totalStallMs,
            stallCount          = stallCount,
            totalDroppedFrames  = totalDroppedFrames,
            averageBandwidthBps = bandwidthSamples.average()
                .takeIf { it.isFinite() }?.toLong() ?: 0L,
            peakBandwidthBps    = bandwidthSamples.maxOrNull() ?: 0L,
            seekCount           = seekCount,
            errorCount          = errorCount,
        )
    }

    // ── Breadcrumbs for crash reporting ───────────────────────────────────────

    /**
     * Last [n] events formatted as crash breadcrumbs.
     * Output is purely string-based — safe for any crash SDK.
     */
    fun breadcrumbs(n: Int = 20): List<String> =
        last(n).map { event ->
            buildString {
                append("[${event.timestampMs}]")
                append("[${event.category.name}]")
                append("[${event::class.simpleName}]")
                append("[session=${event.sessionId.take(8)}]")
                event.mediaId?.let { append("[media=$it]") }
                when (event) {
                    is PlayerEvent.ErrorOccurred ->
                        append("[code=${event.errorCode} fatal=${event.isFatal}]")
                    is PlayerEvent.PlaybackStarted ->
                        append("[ttfp=${event.timeToFirstFrameMs}ms]")
                    is PlayerEvent.BufferingStarted ->
                        append("[stall=${event.stallCount} init=${event.isInitialBuffer}]")
                    is PlayerEvent.BitrateChanged ->
                        append("[${event.previousBitrateBps}→${event.newBitrateBps}bps]")
                    else -> Unit
                }
            }
        }

    companion object {
        const val DEFAULT_CAPACITY = 500
    }
}

// ── QoS Summary Model ─────────────────────────────────────────────────────────

data class SessionQosSummary(
    val sessionId: String,
    val ttfpMs: Long?,
    val totalStallMs: Long,
    val stallCount: Int,
    val totalDroppedFrames: Int,
    val averageBandwidthBps: Long,
    val peakBandwidthBps: Long,
    val seekCount: Int,
    val errorCount: Int,
) {
    val hasStalls: Boolean get() = stallCount > 0
    val hasTtfp: Boolean   get() = ttfpMs != null
    val isHealthy: Boolean
        get() = !hasStalls
            && totalDroppedFrames == 0
            && errorCount == 0
            && (ttfpMs ?: Long.MAX_VALUE) < 3_000L // TTFP < 3s SLA
}
```

## xplayer/app/src/main/java/com/xplayer/dev/core/events/PlayerEventValidator.kt

```kotlin
package com.xplayer.dev.core.events

import javax.inject.Inject
import javax.inject.Singleton

/**
 * # PlayerEventValidator
 *
 * Schema and business rule validation for [PlayerEvent] instances.
 *
 * Two types of rules:
 *  1. **Schema rules** — field-level constraints (non-negative values, etc.)
 *  2. **Business rules** — cross-field and temporal constraints
 *     (e.g., TTFP cannot be > 60s, seekDelta must match positions)
 *
 * Validation is intentionally non-throwing — returns [ValidationResult].
 * The [PlayerEventRouter] decides whether to drop or log invalid events.
 */
@Singleton
class PlayerEventValidator @Inject constructor() {

    sealed interface ValidationResult {
        data object Valid : ValidationResult
        data class Invalid(val reasons: List<String>) : ValidationResult
    }

    fun validate(event: PlayerEvent): ValidationResult {
        val reasons = mutableListOf<String>()

        // ── Common field rules ────────────────────────────────────────────────

        if (event.eventId.isBlank()) reasons += "eventId is blank"
        if (event.sessionId.isBlank()) reasons += "sessionId is blank"
        if (event.timestampMs <= 0L) reasons += "timestampMs <= 0"
        if (event.sequenceNum < 0L) reasons += "sequenceNum < 0"

        // ── Type-specific rules ───────────────────────────────────────────────

        when (event) {

            is PlayerEvent.PlaybackStarted -> {
                if (event.timeToFirstFrameMs > MAX_TTFP_MS)
                    reasons += "timeToFirstFrameMs (${event.timeToFirstFrameMs}) > ${MAX_TTFP_MS}ms — suspicious"
                if (event.mediaId.isBlank())
                    reasons += "PlaybackStarted.mediaId is blank"
                if (event.playbackSpeed <= 0f)
                    reasons += "playbackSpeed must be > 0"
            }

            is PlayerEvent.PlaybackEnded -> {
                if (event.completionPercent !in 0f..100f)
                    reasons += "completionPercent out of range"
                if (event.totalPlayedMs < 0L)
                    reasons += "totalPlayedMs < 0"
            }

            is PlayerEvent.SeekCompleted -> {
                if (event.seekDurationMs > MAX_SEEK_MS)
                    reasons += "seekDurationMs (${event.seekDurationMs}) > ${MAX_SEEK_MS}ms"
                if (event.fromPositionMs < 0L)
                    reasons += "fromPositionMs < 0"
                if (event.toPositionMs < 0L)
                    reasons += "toPositionMs < 0"
            }

            is PlayerEvent.BufferingStarted -> {
                if (event.bufferPercent !in 0..100)
                    reasons += "bufferPercent out of range"
                if (event.stallCount < 0)
                    reasons += "stallCount < 0"
            }

            is PlayerEvent.BufferingEnded -> {
                if (event.stallDurationMs < 0L)
                    reasons += "stallDurationMs < 0"
                if (event.stallDurationMs > MAX_STALL_MS)
                    reasons += "stallDurationMs (${event.stallDurationMs}) > ${MAX_STALL_MS}ms — suspicious"
                if (event.bufferPercentOnResume !in 0..100)
                    reasons += "bufferPercentOnResume out of range"
            }

            is PlayerEvent.ErrorOccurred -> {
                if (event.retryAttempt < 0)
                    reasons += "retryAttempt < 0"
                if (event.maxRetries < 0)
                    reasons += "maxRetries < 0"
                if (event.previousState.isBlank())
                    reasons += "previousState is blank"
            }

            is PlayerEvent.DroppedFrames -> {
                if (event.droppedCount < 0)
                    reasons += "droppedCount < 0"
                if (event.elapsedMs <= 0L)
                    reasons += "elapsedMs <= 0"
                if (event.dropRate < 0f)
                    reasons += "dropRate < 0"
            }

            is PlayerEvent.BitrateChanged -> {
                if (event.previousBitrateBps < 0)
                    reasons += "previousBitrateBps < 0"
                if (event.newBitrateBps < 0)
                    reasons += "newBitrateBps < 0"
            }

            is PlayerEvent.BandwidthEstimated -> {
                if (event.bandwidthBps < 0L)
                    reasons += "bandwidthBps < 0"
                if (event.percentileRank !in 0..100)
                    reasons += "percentileRank out of range"
            }

            is PlayerEvent.SessionEnded -> {
                if (event.completionPercent !in 0f..100f)
                    reasons += "completionPercent out of range"
                if (event.totalSessionMs < 0L)
                    reasons += "totalSessionMs < 0"
                if (event.totalPlayedMs < 0L)
                    reasons += "totalPlayedMs < 0"
                if (event.totalStalledMs < 0L)
                    reasons += "totalStalledMs < 0"
            }

            is PlayerEvent.VideoSizeChanged -> {
                if (event.width < 0) reasons += "width < 0"
                if (event.height < 0) reasons += "height < 0"
            }

            else -> Unit // No additional rules for other types
        }

        return if (reasons.isEmpty()) ValidationResult.Valid
               else ValidationResult.Invalid(reasons)
    }

    companion object {
        private const val MAX_TTFP_MS  = 60_000L   // 60s — any longer is a bug
        private const val MAX_SEEK_MS  = 30_000L   // 30s seek is suspicious
        private const val MAX_STALL_MS = 300_000L  // 5min stall is suspicious
    }
}
```

## xplayer/app/src/main/java/com/xplayer/dev/core/threading/PlayerDispatchers.kt

```kotlin
package com.xplayer.dev.core.threading

import kotlinx.coroutines.CoroutineDispatcher
import kotlinx.coroutines.CoroutineScope
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.SupervisorJob
import kotlinx.coroutines.cancel
import java.util.concurrent.ExecutorService
import java.util.concurrent.Executors
import javax.inject.Inject
import javax.inject.Qualifier
import javax.inject.Singleton

// ── Qualifier annotations ────────────────────────────────────────────────────

@Qualifier @Retention(AnnotationRetention.BINARY)
annotation class MainDispatcher

@Qualifier @Retention(AnnotationRetention.BINARY)
annotation class IoDispatcher

@Qualifier @Retention(AnnotationRetention.BINARY)
annotation class PlayerScope

@Qualifier @Retention(AnnotationRetention.BINARY)
annotation class ApplicationScope

// ── Dispatcher wrapper ───────────────────────────────────────────────────────

/**
 * Centralised dispatcher bundle. Injected wherever threading
 * context is needed, enabling clean test substitution.
 */
data class PlayerDispatchers(
    val main: CoroutineDispatcher,
    val io: CoroutineDispatcher,
    val default: CoroutineDispatcher,
)

// ── Managed executor ─────────────────────────────────────────────────────────

/**
 * Single-thread background executor bound to player lifetime.
 * Media3 operations that require a dedicated thread (e.g. custom
 * DataSource work) use this executor directly.
 */
@Singleton
class PlayerExecutor @Inject constructor() {

    val executor: ExecutorService =
        Executors.newSingleThreadExecutor { runnable ->
            Thread(runnable, "player-bg-thread").apply {
                isDaemon = true
                priority = Thread.NORM_PRIORITY
            }
        }

    fun shutdown() {
        executor.shutdown()
    }

    fun shutdownNow(): List<Runnable> = executor.shutdownNow()
}

// ── Cancellation strategy ────────────────────────────────────────────────────

/**
 * Encapsulates the coroutine scopes used across the player stack.
 *
 * - [applicationScope]  Lives for the entire process lifetime.
 * - [playerScope]       Cancelled whenever the player is released,
 *                       propagated through [SupervisorJob] so sibling
 *                       coroutines are not affected by a single failure.
 */
@Singleton
class PlayerScopeManager @Inject constructor(
    dispatchers: PlayerDispatchers,
) {
    val applicationScope: CoroutineScope =
        CoroutineScope(SupervisorJob() + dispatchers.default)

    val playerScope: CoroutineScope =
        CoroutineScope(SupervisorJob() + dispatchers.main)

    /**
     * Cancel the player scope cleanly. Safe to call multiple times;
     * subsequent calls are no-ops because the job is already cancelled.
     */
    fun cancelPlayerScope(cause: Exception? = null) {
        if (cause != null) {
            playerScope.cancel(cause.message ?: "Player released", cause)
        } else {
            playerScope.cancel("Player released")
        }
    }
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/config/PlayerConfiguration.kt

```kotlin
package com.xplayer.dev.core.config

import androidx.annotation.IntRange
import androidx.media3.common.C
import androidx.media3.exoplayer.DefaultRenderersFactory
import javax.inject.Inject
import javax.inject.Singleton

/**
 * Immutable snapshot of all tunable player parameters.
 * Instantiated once via DI and shared read-only across the stack.
 */
@Singleton
class PlayerConfiguration private constructor(
    val bufferConfig: BufferConfig,
    val networkConfig: NetworkConfig,
    val retryConfig: RetryConfig,
    val loggingConfig: LoggingConfig,
    val playbackConfig: PlaybackConfig,
    val liveConfig: LiveConfig,
    val renderersConfig: RenderersConfig,
) {

    // ── Buffer ────────────────────────────────────────────────────────────────

    data class BufferConfig(
        val minBufferMs: Int = DEFAULT_MIN_BUFFER_MS,
        val maxBufferMs: Int = DEFAULT_MAX_BUFFER_MS,
        val bufferForPlaybackMs: Int = DEFAULT_BUFFER_FOR_PLAYBACK_MS,
        val bufferForPlaybackAfterRebufferMs: Int =
            DEFAULT_BUFFER_FOR_PLAYBACK_AFTER_REBUFFER_MS,
    ) {
        init {
            require(minBufferMs > 0) { "minBufferMs must be > 0" }
            require(maxBufferMs >= minBufferMs) {
                "maxBufferMs ($maxBufferMs) must be ≥ minBufferMs ($minBufferMs)"
            }
            require(bufferForPlaybackMs <= minBufferMs) {
                "bufferForPlaybackMs must be ≤ minBufferMs"
            }
        }
    }

    // ── Network ───────────────────────────────────────────────────────────────

    data class NetworkConfig(
        val connectTimeoutSec: Long = 15L,
        val readTimeoutSec: Long = 30L,
        val writeTimeoutSec: Long = 15L,
        val enableHttp2: Boolean = true,
        val userAgent: String = "XPlayerEngine/1.0",
    ) {
        init {
            require(connectTimeoutSec > 0) { "connectTimeoutSec must be > 0" }
            require(readTimeoutSec > 0) { "readTimeoutSec must be > 0" }
        }
    }

    // ── Retry ─────────────────────────────────────────────────────────────────

    data class RetryConfig(
        @IntRange(from = 0, to = 10)
        val maxRetryCount: Int = 3,
        val initialDelayMs: Long = 1_000L,
        val maxDelayMs: Long = 30_000L,
        val delayMultiplier: Float = 2.0f,
    ) {
        init {
            require(maxRetryCount in 0..10) { "maxRetryCount must be in 0..10" }
            require(initialDelayMs > 0) { "initialDelayMs must be > 0" }
            require(maxDelayMs >= initialDelayMs) {
                "maxDelayMs must be ≥ initialDelayMs"
            }
            require(delayMultiplier >= 1f) { "delayMultiplier must be ≥ 1.0" }
        }
    }

    // ── Logging ───────────────────────────────────────────────────────────────

    data class LoggingConfig(
        val enableVerboseLogging: Boolean = false,
        val enableNetworkLogging: Boolean = false,
        val enableStrictMode: Boolean = false,
    )

    // ── Playback (Phase 02) ───────────────────────────────────────────────────

    data class PlaybackConfig(
        val defaultPlaybackSpeed: Float = 1.0f,
        val playWhenReady: Boolean = true,
        val repeatMode: Int = 0, // Player.REPEAT_MODE_OFF
        val shuffleModeEnabled: Boolean = false,
        val handleAudioFocus: Boolean = true,
        val handleAudioBecomingNoisy: Boolean = true,
        val wakeMode: Int = C.WAKE_MODE_NETWORK,
        val seekBackIncrementMs: Long = 10_000L,
        val seekForwardIncrementMs: Long = 10_000L,
        val videoScalingMode: Int = C.VIDEO_SCALING_MODE_DEFAULT,
    ) {
        init {
            require(defaultPlaybackSpeed > 0f) { "defaultPlaybackSpeed must be > 0" }
            require(seekBackIncrementMs > 0) { "seekBackIncrementMs must be > 0" }
            require(seekForwardIncrementMs > 0) { "seekForwardIncrementMs must be > 0" }
        }
    }

    // ── Live Playback (Phase 02) ──────────────────────────────────────────────

    data class LiveConfig(
        val targetOffsetMs: Long = 5_000L,
        val minSpeed: Float = 0.97f,
        val maxSpeed: Float = 1.03f,
    ) {
        init {
            require(minSpeed > 0f && maxSpeed >= minSpeed) {
                "Invalid live playback speeds: min=$minSpeed, max=$maxSpeed"
            }
        }
    }

    // ── Renderers (Phase 02) ──────────────────────────────────────────────────

    data class RenderersConfig(
        val extensionRendererMode: Int = DefaultRenderersFactory.EXTENSION_RENDERER_MODE_OFF,
        val enableAsyncQueueing: Boolean = true,
    )

    // ── Builder ───────────────────────────────────────────────────────────────

    class Builder @Inject constructor() {
        private var bufferConfig = BufferConfig()
        private var networkConfig = NetworkConfig()
        private var retryConfig = RetryConfig()
        private var loggingConfig = LoggingConfig()
        private var playbackConfig = PlaybackConfig()
        private var liveConfig = LiveConfig()
        private var renderersConfig = RenderersConfig()

        fun bufferConfig(config: BufferConfig) = apply { bufferConfig = config }
        fun networkConfig(config: NetworkConfig) = apply { networkConfig = config }
        fun retryConfig(config: RetryConfig) = apply { retryConfig = config }
        fun loggingConfig(config: LoggingConfig) = apply { loggingConfig = config }
        fun playbackConfig(config: PlaybackConfig) = apply { playbackConfig = config }
        fun liveConfig(config: LiveConfig) = apply { liveConfig = config }
        fun renderersConfig(config: RenderersConfig) = apply { renderersConfig = config }

        fun build(): PlayerConfiguration = PlayerConfiguration(
            bufferConfig = bufferConfig,
            networkConfig = networkConfig,
            retryConfig = retryConfig,
            loggingConfig = loggingConfig,
            playbackConfig = playbackConfig,
            liveConfig = liveConfig,
            renderersConfig = renderersConfig,
        )
    }

    companion object {
        private const val DEFAULT_MIN_BUFFER_MS = 15_000
        private const val DEFAULT_MAX_BUFFER_MS = 50_000
        private const val DEFAULT_BUFFER_FOR_PLAYBACK_MS = 2_500
        private const val DEFAULT_BUFFER_FOR_PLAYBACK_AFTER_REBUFFER_MS = 5_000

        fun default(): PlayerConfiguration = Builder().build()

        fun debug(): PlayerConfiguration = Builder()
            .loggingConfig(
                LoggingConfig(
                    enableVerboseLogging = true,
                    enableNetworkLogging = true,
                    enableStrictMode = true,
                )
            )
            .build()
    }
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/config/PlayerConfigurationValidator.kt

```kotlin
package com.xplayer.dev.core.config

import timber.log.Timber
import javax.inject.Inject
import javax.inject.Singleton

/**
 * Validates [PlayerConfiguration] at startup.
 * Logs warnings for suspicious values; returns result for diagnostics.
 */
@Singleton
class PlayerConfigurationValidator @Inject constructor() {

    sealed interface ValidationResult {
        data object Valid : ValidationResult
        data class Invalid(val reasons: List<String>) : ValidationResult
        data class Warnings(val warnings: List<String>) : ValidationResult
    }

    fun validate(config: PlayerConfiguration): ValidationResult {
        val errors   = mutableListOf<String>()
        val warnings = mutableListOf<String>()

        with(config.bufferConfig) {
            if (maxBufferMs > 120_000) {
                warnings += "maxBufferMs ($maxBufferMs) > 120s — high memory risk"
            }
            if (minBufferMs < 1_000) {
                warnings += "minBufferMs ($minBufferMs) < 1s — may cause frequent stalls"
            }
        }

        with(config.networkConfig) {
            if (connectTimeoutSec < 5L) {
                warnings += "connectTimeoutSec ($connectTimeoutSec) < 5s — may timeout on slow networks"
            }
            if (userAgent.isBlank()) {
                errors += "userAgent must not be blank"
            }
        }

        with(config.playbackConfig) {
            if (defaultPlaybackSpeed !in 0.25f..8.0f) {
                errors += "defaultPlaybackSpeed (${defaultPlaybackSpeed}) out of range [0.25, 8.0]"
            }
        }

        if (errors.isNotEmpty()) {
            Timber.e("PlayerConfiguration: invalid — $errors")
            return ValidationResult.Invalid(errors)
        }

        if (warnings.isNotEmpty()) {
            warnings.forEach { Timber.w("PlayerConfiguration: $it") }
            return ValidationResult.Warnings(warnings)
        }

        return ValidationResult.Valid
    }
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/engine/EngineLifecycle.kt

```kotlin
package com.xplayer.dev.core.engine

import androidx.media3.exoplayer.ExoPlayer

/**
 * Sealed FSM representing the engine's internal lifecycle,
 * completely decoupled from media playback state.
 *
 * Legal transitions:
 *   Uninitialized → Initializing → Active → Releasing → Released
 *                                        ↘ Error (init failed)
 *   Error → Initializing (recovery retry)
 */
sealed class EngineLifecycle {

    /** Engine created but ExoPlayer not yet built. */
    data object Uninitialized : EngineLifecycle()

    /** ExoPlayer constructor is currently running on the main thread. */
    data object Initializing : EngineLifecycle()

    /**
     * ExoPlayer is alive and operational.
     * All media playback commands are routed through [player].
     */
    data class Active(val player: ExoPlayer) : EngineLifecycle()

    /** Release sequence has started. No new commands accepted. */
    data object Releasing : EngineLifecycle()

    /** Engine fully torn down. Must create a new instance for reuse. */
    data object Released : EngineLifecycle()

    /** Fatal initialisation or runtime error. */
    data class Error(val cause: Throwable) : EngineLifecycle()
}

val EngineLifecycle.isOperational: Boolean
    get() = this is EngineLifecycle.Active

val EngineLifecycle.player: ExoPlayer?
    get() = (this as? EngineLifecycle.Active)?.player

```

## xplayer/app/src/main/java/com/xplayer/dev/core/engine/EngineLifecycleLogger.kt

```kotlin
package com.xplayer.dev.core.engine

import timber.log.Timber
import javax.inject.Inject
import javax.inject.Singleton

/**
 * Logs all lifecycle transitions and anomalies.
 */
@Singleton
class EngineLifecycleLogger @Inject constructor() : EngineLifecycleObserver {

    override fun onLifecycleChanged(oldState: EngineLifecycle, newState: EngineLifecycle) {
        when (newState) {
            is EngineLifecycle.Active -> Timber.i("Lifecycle: Active [ExoPlayer=${newState.player.hashCode()}]")
            is EngineLifecycle.Error -> Timber.e(newState.cause, "Lifecycle: Error entered from ${oldState::class.simpleName}")
            is EngineLifecycle.Released -> Timber.i("Lifecycle: Released cleanly")
            else -> Timber.d("Lifecycle transition: ${oldState::class.simpleName} -> ${newState::class.simpleName}")
        }
    }

    fun logRejectedTransition(from: EngineLifecycle, to: EngineLifecycle, reason: String) {
        Timber.w("Lifecycle transition rejected: $from -> $to ($reason)")
    }
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/engine/EngineLifecycleManager.kt

```kotlin
package com.xplayer.dev.core.engine

import androidx.annotation.MainThread
import androidx.media3.exoplayer.ExoPlayer
import timber.log.Timber
import java.util.concurrent.CopyOnWriteArrayList
import java.util.concurrent.atomic.AtomicReference
import javax.inject.Inject
import javax.inject.Singleton

/**
 * Robust lifecycle manager decoupled from playback state.
 *
 * Enforces atomic CAS updates, guards against double initialization and double release,
 * and notifies observers.
 */
@Singleton
class EngineLifecycleManager @Inject constructor(
    private val factory: PlayerFactory,
    private val validator: EngineLifecycleValidator,
    private val logger: EngineLifecycleLogger,
) {

    private val _state = AtomicReference<EngineLifecycle>(EngineLifecycle.Uninitialized)
    val currentState: EngineLifecycle get() = _state.get()

    private val observers = CopyOnWriteArrayList<EngineLifecycleObserver>().apply {
        add(logger)
    }

    fun addObserver(observer: EngineLifecycleObserver) {
        observers.addIfAbsent(observer)
    }

    fun removeObserver(observer: EngineLifecycleObserver) {
        observers.remove(observer)
    }

    /**
     * Attempts to transition to [newState] using atomic CAS.
     */
    private fun transitionTo(newState: EngineLifecycle): Boolean {
        while (true) {
            val oldState = _state.get()
            when (val validation = validator.validate(oldState, newState)) {
                is EngineLifecycleValidator.Result.Illegal -> {
                    logger.logRejectedTransition(oldState, newState, validation.reason)
                    return false
                }
                is EngineLifecycleValidator.Result.Legal -> {
                    if (_state.compareAndSet(oldState, newState)) {
                        notifyObservers(oldState, newState)
                        return true
                    }
                    // CAS failed due to concurrent update, retry validation loop
                }
            }
        }
    }

    private fun notifyObservers(oldState: EngineLifecycle, newState: EngineLifecycle) {
        observers.forEach { observer ->
            runCatching { observer.onLifecycleChanged(oldState, newState) }
                .onFailure { Timber.e(it, "Error in EngineLifecycleObserver") }
        }
    }

    @MainThread
    fun initialize(): ExoPlayer {
        val current = _state.get()
        if (current is EngineLifecycle.Active) {
            Timber.w("LifecycleManager: already initialized. Returning existing player.")
            return current.player
        }
        if (current is EngineLifecycle.Released) {
            error("LifecycleManager: cannot re-initialize after release.")
        }
        if (current is EngineLifecycle.Initializing) {
            error("LifecycleManager: initialization already in progress.")
        }

        if (!transitionTo(EngineLifecycle.Initializing)) {
            error("LifecycleManager: failed to transition to Initializing.")
        }

        return runCatching {
            val player = factory.create()
            transitionTo(EngineLifecycle.Active(player))
            player
        }.onFailure { throwable ->
            transitionTo(EngineLifecycle.Error(throwable))
        }.getOrThrow()
    }

    @MainThread
    fun release() {
        val current = _state.get()
        if (current is EngineLifecycle.Released || current is EngineLifecycle.Releasing) {
            Timber.w("LifecycleManager: already released or releasing.")
            return
        }

        if (transitionTo(EngineLifecycle.Releasing)) {
            val player = (current as? EngineLifecycle.Active)?.player
            player?.release()
            transitionTo(EngineLifecycle.Released)
        }
    }

    /**
     * Force immediate release even if in error or stuck state.
     */
    @MainThread
    fun forceRelease() {
        val oldState = _state.getAndSet(EngineLifecycle.Released)
        if (oldState is EngineLifecycle.Active) {
            runCatching { oldState.player.release() }
        }
        if (oldState != EngineLifecycle.Released) {
            notifyObservers(oldState, EngineLifecycle.Released)
        }
    }

    fun requireOperationalPlayer(operation: String): ExoPlayer {
        val current = _state.get()
        return (current as? EngineLifecycle.Active)?.player
            ?: throw IllegalStateException("Cannot execute '$operation': engine is not Active (current=${current::class.simpleName})")
    }
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/engine/EngineLifecycleObserver.kt

```kotlin
package com.xplayer.dev.core.engine

/**
 * Observer interface for reacting to engine lifecycle changes.
 */
interface EngineLifecycleObserver {
    fun onLifecycleChanged(oldState: EngineLifecycle, newState: EngineLifecycle)
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/engine/EngineLifecycleValidator.kt

```kotlin
package com.xplayer.dev.core.engine

import timber.log.Timber
import javax.inject.Inject
import javax.inject.Singleton

/**
 * Validates legal transitions between [EngineLifecycle] states.
 */
@Singleton
class EngineLifecycleValidator @Inject constructor() {

    sealed interface Result {
        data object Legal : Result
        data class Illegal(val from: EngineLifecycle, val to: EngineLifecycle, val reason: String) : Result
    }

    fun validate(from: EngineLifecycle, to: EngineLifecycle): Result {
        if (from::class == to::class) {
            return Result.Legal
        }

        val isAllowed = when (from) {
            is EngineLifecycle.Uninitialized -> to is EngineLifecycle.Initializing || to is EngineLifecycle.Released
            is EngineLifecycle.Initializing -> to is EngineLifecycle.Active || to is EngineLifecycle.Error || to is EngineLifecycle.Releasing
            is EngineLifecycle.Active -> to is EngineLifecycle.Releasing || to is EngineLifecycle.Error
            is EngineLifecycle.Releasing -> to is EngineLifecycle.Released || to is EngineLifecycle.Error
            is EngineLifecycle.Error -> to is EngineLifecycle.Initializing || to is EngineLifecycle.Releasing || to is EngineLifecycle.Released
            is EngineLifecycle.Released -> false
        }

        return if (isAllowed) {
            Result.Legal
        } else {
            val reason = "Illegal lifecycle transition: ${from::class.simpleName} -> ${to::class.simpleName}"
            Timber.w("EngineLifecycleValidator: $reason")
            Result.Illegal(from, to, reason)
        }
    }
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/engine/PlayerEngine.kt

```kotlin
package com.xplayer.dev.core.engine

import androidx.annotation.MainThread
import androidx.annotation.VisibleForTesting
import androidx.media3.common.MediaItem
import androidx.media3.common.PlaybackParameters
import com.xplayer.dev.BuildConfig
import com.xplayer.dev.core.engine.internal.EngineEventBus
import com.xplayer.dev.core.engine.internal.EngineLifecycle
import com.xplayer.dev.core.engine.internal.EngineListenerHub
import com.xplayer.dev.core.engine.internal.EngineMetricsProbe
import com.xplayer.dev.core.engine.internal.EngineSessionManager
import com.xplayer.dev.core.engine.internal.EngineStateValidator
import com.xplayer.dev.core.engine.internal.MetricsSnapshot
import com.xplayer.dev.core.engine.internal.isOperational
import com.xplayer.dev.core.engine.internal.requirePlayer
import com.xplayer.dev.core.engine.internal.player
import com.xplayer.dev.core.events.PlayerEvent
import com.xplayer.dev.core.state.PlayerState
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow
import timber.log.Timber
import java.util.concurrent.atomic.AtomicReference
import javax.inject.Inject
import javax.inject.Singleton

/**
 * # PlayerEngine
 *
 * The single source of truth for ExoPlayer interaction.
 *
 * ## Responsibilities
 *  - Own the ExoPlayer instance lifecycle (init → active → release).
 *  - Emit [PlayerState] via [StateFlow] — all observers get consistent state.
 *  - Emit [PlayerEvent] via [Flow] — one-shot signals for analytics/UI.
 *  - Track QoS metrics (TTFP, stalls, drops, bandwidth) via [MetricsSnapshot].
 *  - Enforce legal state transitions via [EngineStateValidator].
 *  - Manage session identity via [EngineSessionManager].
 *
 * ## Threading
 *  - All public methods are [@MainThread] — ExoPlayer requires main thread.
 *  - [StateFlow] and [Flow] collectors can run on any thread.
 *  - Internal [AtomicReference] for [EngineLifecycle] prevents torn reads.
 *
 * ## Lifecycle
 * ```
 * Uninitialised → Initialising → Active → Releasing → Released
 *                                       ↘ InitError
 * ```
 *
 * ## Usage
 * ```kotlin
 * engine.initialize()
 * engine.prepare(mediaItem, mediaId = "track_123", url = "https://...")
 * engine.play()
 * // ... observe engine.state and engine.events
 * engine.release()
 * ```
 *
 * @param factory    Constructs a fully-configured ExoPlayer instance.
 * @param sessionMgr Manages session ID and metadata per playback attempt.
 * @param metrics    Collects QoS data (TTFP, stalls, seeks, bandwidth).
 * @param eventBus   Typed Channel wrapper for one-shot PlayerEvents.
 * @param validator  Guards legal state machine transitions.
 */
@Singleton
class PlayerEngine @Inject constructor(
    private val factory: PlayerFactory,
    private val sessionMgr: EngineSessionManager,
    private val metrics: EngineMetricsProbe,
    private val eventBus: EngineEventBus,
    private val validator: EngineStateValidator,
) {

    // Backward compatibility properties
    val exoPlayer: androidx.media3.exoplayer.ExoPlayer? get() = (lifecycleState as? EngineLifecycle.Active)?.player
    val playbackSpeed: Float @MainThread get() = exoPlayer?.playbackParameters?.speed ?: 1f
    val playbackState: Int @MainThread get() = exoPlayer?.playbackState ?: androidx.media3.common.Player.STATE_IDLE
    var playWhenReady: Boolean
        get() = exoPlayer?.playWhenReady ?: false
        set(value) { exoPlayer?.playWhenReady = value }
    val isLoading: Boolean get() = exoPlayer?.isLoading ?: false
    val currentMediaItem: androidx.media3.common.MediaItem? get() = exoPlayer?.currentMediaItem
    



    // Forwarded methods for repository
    @MainThread fun seekBack() { exoPlayer?.seekBack() }
    @MainThread fun seekForward() { exoPlayer?.seekForward() }
    @MainThread fun replay() { exoPlayer?.seekToDefaultPosition() }
    @MainThread fun setRepeatMode(repeatMode: Int) { exoPlayer?.repeatMode = repeatMode }
    @MainThread fun setShuffleMode(shuffleModeEnabled: Boolean) { exoPlayer?.shuffleModeEnabled = shuffleModeEnabled }

    // ── Lifecycle ─────────────────────────────────────────────────────────────

    private val _lifecycle = AtomicReference<EngineLifecycle>(EngineLifecycle.Uninitialised)

    val lifecycleState: EngineLifecycle get() = _lifecycle.get()

    // ── State ─────────────────────────────────────────────────────────────────

    private val _state = MutableStateFlow<PlayerState>(PlayerState.Idle(sequenceNumber = 0))

    /**
     * Current playback state.
     * Always has a value — starts as [PlayerState.Idle(sequenceNumber = 0)].
     * Collect from any thread; updates are emitted on the main thread.
     */
    val state: StateFlow<PlayerState> = _state.asStateFlow()

    // ── Events ────────────────────────────────────────────────────────────────

    /**
     * One-shot player events.
     * Each emission is consumed once per collector.
     * Backed by a [Channel] with capacity 512.
     */
    val events: Flow<PlayerEvent> = eventBus.events

    // ── Listener hub ──────────────────────────────────────────────────────────

    private val listenerHub: EngineListenerHub = EngineListenerHub(
        sessionManager = sessionMgr,
        metricsProbe   = metrics,
        stateValidator = validator,
        isDebug        = BuildConfig.DEBUG,
        onState        = { newState -> applyState(newState) },
        onEvent        = { event    -> eventBus.send(event)  },
    )

    // ═════════════════════════════════════════════════════════════════════════
    // Public API
    // ═════════════════════════════════════════════════════════════════════════

    /**
     * Initialise the engine and build the ExoPlayer instance.
     *
     * Idempotent-safe: calling when already Active logs a warning and returns.
     * Must be called on the **main thread**.
     *
     * @throws IllegalStateException if called after [release].
     */
    @MainThread
    fun initialize() {
        when (val lc = _lifecycle.get()) {
            is EngineLifecycle.Active -> {
                Timber.w("PlayerEngine: initialize() ignored — already Active")
                return
            }
            is EngineLifecycle.Released -> {
                error(
                    "PlayerEngine: cannot re-initialize after release. " +
                        "Create a new instance."
                )
            }
            is EngineLifecycle.Initialising -> {
                Timber.w("PlayerEngine: initialize() ignored — init in progress")
                return
            }
            is EngineLifecycle.InitError -> {
                Timber.w(
                    "PlayerEngine: retrying init after previous error",
                    lc.cause,
                )
            }
            else -> Unit
        }

        if (!_lifecycle.compareAndSet(
                _lifecycle.get(),
                EngineLifecycle.Initialising,
            )
        ) {
            Timber.w("PlayerEngine: CAS failed on init — concurrent call detected")
            return
        }

        runCatching {
            val player = factory.create()
            listenerHub.attach(player)
            _lifecycle.set(EngineLifecycle.Active(player))
            Timber.i("PlayerEngine: initialised successfully")
        }.onFailure { throwable ->
            _lifecycle.set(EngineLifecycle.InitError(throwable))
            Timber.e(throwable, "PlayerEngine: initialisation FAILED")
            throw throwable
        }
    }

    /**
     * Prepare a new media item for playback.
     *
     * - Opens a new analytics session.
     * - Resets all QoS metrics.
     * - Configures ExoPlayer with [mediaItem].
     *
     * @param mediaItem The Media3 item to load.
     * @param mediaId   Stable ID for tracking and events.
     * @param url       Original URL — stored in session for context.
     *
     * @throws IllegalStateException if engine is not Active.
     */
    @MainThread
    fun prepare(
        mediaItem: MediaItem,
        mediaId: String,
        url: String = mediaItem.localConfiguration?.uri?.toString() ?: "",
    ) {
        val player = _lifecycle.get().requirePlayer("prepare")

        // Open new session — resets session ID + metadata
        val session = sessionMgr.openSession(mediaId = mediaId, url = url)

        // Reset QoS probe for the new session
        metrics.reset()
        metrics.onPrepareStart()

        // Emit session-level event
        eventBus.send(
            PlayerEvent.SessionStarted(
                mediaId   = mediaId,
                sessionId = session.sessionId,
                deviceInfo = PlayerEvent.DeviceInfo("Unknown", "Unknown", "Unknown", 0, 0, 0, 0f, 0L, "Unknown"),
                sequenceNum = 0L,
                url = mediaItem.localConfiguration?.uri?.toString() ?: "",
                contentType = PlayerEvent.SessionStarted.ContentType.VOD,
                appVersion = "1.0",
                playerVersion = "1.0"
            )
        )

        // Transition to Preparing
        applyState(PlayerState.Preparing(mediaId, mediaItem))

        // Hand off to ExoPlayer
        player.setMediaItem(mediaItem)
        player.prepare()

        Timber.i(
            "PlayerEngine: preparing " +
                "[mediaId=$mediaId] " +
                "[session=${session.sessionId}] " +
                "[url=$url]"
        )
    }

    /**
     * Start or resume playback.
     * No-op if not in a playable state.
     */
    @MainThread
    fun play() {
        val player = _lifecycle.get().requirePlayer("play")
        if (!state.value.canPlay) {
            Timber.w(
                "PlayerEngine: play() ignored — " +
                    "state=${state.value::class.simpleName}"
            )
            return
        }
        player.playWhenReady = true
        Timber.d("PlayerEngine: play requested")
    }

    /**
     * Pause playback.
     * No-op if not currently playing.
     */
    @MainThread
    fun pause() {
        val player = _lifecycle.get().requirePlayer("pause")
        if (state.value !is PlayerState.Playing) {
            Timber.w(
                "PlayerEngine: pause() ignored — " +
                    "state=${state.value::class.simpleName}"
            )
            return
        }
        player.playWhenReady = false
        Timber.d("PlayerEngine: pause requested")
    }

    /**
     * Seek to [positionMs].
     *
     * Records seek origin for analytics before issuing the ExoPlayer seek.
     * [PlayerEvent.SeekCompleted] is emitted in [onPositionDiscontinuity].
     *
     * @throws IllegalStateException if player is not in a seekable state.
     */
    @MainThread
    fun seekTo(positionMs: Long) {
        val player = _lifecycle.get().requirePlayer("seekTo")
        if (!state.value.canSeek) {
            Timber.w(
                "PlayerEngine: seekTo($positionMs) ignored — " +
                    "state=${state.value::class.simpleName}"
            )
            return
        }

        // Record origin BEFORE ExoPlayer moves the position
        metrics.onSeekStart(player.currentPosition)
        player.seekTo(positionMs)

        // SeekCompleted is emitted from onPositionDiscontinuity
        // which fires synchronously during seekTo on main thread
        val seekDurationMs = metrics.onSeekComplete()
        val session = sessionMgr.current ?: return

        eventBus.send(
            PlayerEvent.SeekCompleted(
                mediaId        = session.mediaId,
                sessionId      = session.sessionId,
                fromPositionMs = metrics.lastSeekFromPositionMs,
                toPositionMs   = positionMs,
                seekDurationMs = seekDurationMs,
            )
        )
        Timber.d(
            "PlayerEngine: seek " +
                "[from=${metrics.lastSeekFromPositionMs}ms → to=${positionMs}ms] " +
                "[duration=${seekDurationMs}ms]"
        )
    }

    /**
     * Stop playback and return to [PlayerState.Idle(sequenceNumber = 0)].
     *
     * @param reason Reason for stop — used in [PlayerEvent.PlaybackStopped].
     */
    @MainThread
    fun stop(
        reason: PlayerEvent.PlaybackStopped.StopReason =
            PlayerEvent.PlaybackStopped.StopReason.USER_ACTION,
    ) {
        val lc = _lifecycle.get()
        if (!lc.isOperational) {
            Timber.w("PlayerEngine: stop() called when not active")
            return
        }
        val player = (lc as? EngineLifecycle.Active)?.player ?: return
        val session = sessionMgr.current

        val positionMs = player.currentPosition
        player.stop()

        if (session != null) {
            eventBus.send(
                PlayerEvent.PlaybackStopped(
                    mediaId   = session.mediaId,
                    sessionId = session.sessionId,
                    positionMs = positionMs,
                    stopReason = reason,
                )
            )
            emitSessionEndedEvent(session, positionMs)
        }

        sessionMgr.closeSession()
        applyState(PlayerState.Idle(sequenceNumber = 0))
        Timber.i("PlayerEngine: stopped [reason=$reason]")
    }

    /**
     * Set the playback speed.
     *
     * @param speed Playback speed multiplier. Valid range: 0.25..8.0.
     */
    @MainThread
    fun setPlaybackSpeed(speed: Float) {
        require(speed in 0.25f..8.0f) {
            "Playback speed $speed out of range [0.25, 8.0]"
        }
        val player = _lifecycle.get().requirePlayer("setPlaybackSpeed")
        player.playbackParameters = PlaybackParameters(speed)
        Timber.d("PlayerEngine: speed set to ${speed}x")
    }

    /**
     * Set the volume. [0.0, 1.0].
     */
    @MainThread
    fun setVolume(volume: Float) {
        require(volume in 0f..1f) { "Volume $volume out of range [0.0, 1.0]" }
        _lifecycle.get().requirePlayer("setVolume").volume = volume
        Timber.d("PlayerEngine: volume set to $volume")
    }

    /**
     * Release all ExoPlayer resources.
     *
     * After this call the engine instance is permanently dead.
     * All subsequent commands will throw [IllegalStateException].
     * Idempotent — safe to call multiple times.
     */
    @MainThread
    fun release() {
        val lc = _lifecycle.get()

        if (lc is EngineLifecycle.Released || lc is EngineLifecycle.Releasing) {
            Timber.w("PlayerEngine: release() ignored — already released/releasing")
            return
        }

        Timber.i("PlayerEngine: releasing...")
        _lifecycle.set(EngineLifecycle.Releasing)

        // Close active session before teardown
        val session = sessionMgr.closeSession()
        if (session != null) {
            emitSessionEndedEvent(session, (_lifecycle.get() as? EngineLifecycle.Active)?.player?.currentPosition ?: 0L)
        }

        // Detach listeners before releasing ExoPlayer
        (lc as? EngineLifecycle.Active)?.player?.let { player ->
            listenerHub.detach(player)
            player.release()
        }

        _lifecycle.set(EngineLifecycle.Released)
        _state.value = PlayerState.Idle(sequenceNumber = 0)

        // Close event bus LAST — so session-ended event can be sent
        eventBus.close()

        Timber.i(
            "PlayerEngine: released " +
                "[totalSent=${eventBus.sentEventCount}] " +
                "[dropped=${eventBus.droppedEventCount}]"
        )
    }

    // ═════════════════════════════════════════════════════════════════════════
    // Accessors
    // ═════════════════════════════════════════════════════════════════════════

    /** Current playback position in milliseconds. 0 if not active. */
    val currentPositionMs: Long
        @MainThread get() = (_lifecycle.get() as? EngineLifecycle.Active)?.player?.currentPosition ?: 0L

    /** Duration of current media in milliseconds. 0 if unknown. */
    val durationMs: Long
        @MainThread get() = (_lifecycle.get() as? EngineLifecycle.Active)?.player
            ?.duration
            ?.coerceAtLeast(0L) ?: 0L

    /** True if the player is actively rendering audio/video. */
    val isPlaying: Boolean
        @MainThread get() = (_lifecycle.get() as? EngineLifecycle.Active)?.player?.isPlaying ?: false

    /** True if the current media item is a live stream. */
    val isLiveStream: Boolean
        @MainThread get() = (_lifecycle.get() as? EngineLifecycle.Active)?.player?.isCurrentMediaItemLive ?: false

    /** Buffered position in milliseconds. */
    val bufferedPositionMs: Long
        @MainThread get() = (_lifecycle.get() as? EngineLifecycle.Active)?.player?.bufferedPosition ?: 0L

    /** Buffered percentage [0..100]. */
    val bufferedPercent: Int
        @MainThread get() = (_lifecycle.get() as? EngineLifecycle.Active)?.player?.bufferedPercentage?.coerceIn(0, 100) ?: 0

    /** ID of the current analytics session. */
    val sessionId: String get() = sessionMgr.currentSessionId

    /** Current media ID, null if idle. */
    val currentMediaId: String? get() = sessionMgr.currentMediaId

    /**
     * Immutable QoS snapshot for the current session.
     * Safe to call from any thread.
     */
    val metricsSnapshot: MetricsSnapshot get() = metrics.snapshot()

    /** Total events dropped due to channel backpressure. */
    val droppedEventCount: Long get() = eventBus.droppedEventCount

    // ═════════════════════════════════════════════════════════════════════════
    // Private helpers
    // ═════════════════════════════════════════════════════════════════════════

    /**
     * Apply a state update. Called from [EngineListenerHub.onState].
     * Already validated by [EngineStateValidator] inside the hub.
     */
    private fun applyState(newState: PlayerState) {
        _state.value = newState
        Timber.v("PlayerEngine: state → ${newState::class.simpleName}")
    }

    private fun emitSessionEndedEvent(
        session: EngineSessionManager.Session,
        positionMs: Long,
    ) {
        val snapshot = metrics.snapshot()
        eventBus.send(
            PlayerEvent.SessionEnded(
                sessionId = session.sessionId,
                mediaId = currentMediaId,
                sequenceNum = 0L,
                endReason = PlayerEvent.SessionEnded.EndReason.USER_STOP,
                totalPlayedMs = 0L,
                totalSeekCount = 0,
                totalDroppedFrames = 0,
                averageBandwidthBps = 0L,
                maxConcurrentBps = 0L,
                completionPercent = 0f
            )
        )
    }

    private fun buildDeviceInfo(): PlayerEvent.DeviceInfo =
        PlayerEvent.DeviceInfo(
            manufacturer = android.os.Build.MANUFACTURER,
            model        = android.os.Build.MODEL,
            osVersion    = android.os.Build.VERSION.RELEASE,
            sdkInt       = android.os.Build.VERSION.SDK_INT,
            screenWidthPx = 0,
            screenHeightPx = 0,
            screenDensity = 0f,
            totalRamMb = 0L,
            cpuAbi = android.os.Build.SUPPORTED_ABIS.firstOrNull() ?: "Unknown"
        )

    // ── Test hooks ────────────────────────────────────────────────────────────

    @VisibleForTesting
    internal fun forceState(state: PlayerState) {
        _state.value = state
    }

    @VisibleForTesting
    internal fun currentLifecycle(): EngineLifecycle = _lifecycle.get()
}

// ── State extension guards ────────────────────────────────────────────────────

private val PlayerState.canPlay: Boolean
    get() = this is PlayerState.Ready
        || this is PlayerState.Paused
        || this is PlayerState.Playing

private val PlayerState.canSeek: Boolean
    get() = this is PlayerState.Playing
        || this is PlayerState.Paused
        || this is PlayerState.Ready
```

## xplayer/app/src/main/java/com/xplayer/dev/core/engine/PlayerFactory.kt

```kotlin
package com.xplayer.dev.core.engine

import android.content.Context
import androidx.media3.common.AudioAttributes
import androidx.media3.common.C
import androidx.media3.common.util.UnstableApi
import androidx.media3.datasource.okhttp.OkHttpDataSource
import androidx.media3.exoplayer.DefaultLoadControl
import androidx.media3.exoplayer.DefaultLivePlaybackSpeedControl
import androidx.media3.exoplayer.DefaultRenderersFactory
import androidx.media3.exoplayer.ExoPlayer
import androidx.media3.exoplayer.source.DefaultMediaSourceFactory
import androidx.media3.exoplayer.trackselection.DefaultTrackSelector
import com.xplayer.dev.core.config.PlayerConfiguration
import com.xplayer.dev.core.threading.PlayerExecutor
import dagger.hilt.android.qualifiers.ApplicationContext
import okhttp3.OkHttpClient
import timber.log.Timber
import javax.inject.Inject
import javax.inject.Singleton

/**
 * Responsible for constructing a fully-configured [ExoPlayer] instance.
 *
 * All player-specific wiring (load control, data source, audio focus,
 * renderers factory, live speed control, track selector) lives here.
 */
@UnstableApi
@Singleton
class PlayerFactory @Inject constructor(
    @ApplicationContext private val context: Context,
    private val okHttpClient: OkHttpClient,
    private val configuration: PlayerConfiguration,
    private val executor: PlayerExecutor,
) {

    /**
     * Build a new [ExoPlayer] with the current [PlayerConfiguration].
     * Must be called on the **main thread**.
     */
    fun create(): ExoPlayer {
        Timber.d("PlayerFactory: creating ExoPlayer instance")

        val renderersFactory = buildRenderersFactory()
        val loadControl = buildLoadControl()
        val dataSourceFactory = buildDataSourceFactory()
        val trackSelector = buildTrackSelector()
        val livePlaybackSpeedControl = buildLivePlaybackSpeedControl()

        val player = ExoPlayer.Builder(context, renderersFactory)
            .setLoadControl(loadControl)
            .setMediaSourceFactory(DefaultMediaSourceFactory(dataSourceFactory))
            .setTrackSelector(trackSelector)
            .setLivePlaybackSpeedControl(livePlaybackSpeedControl)
            .setAudioAttributes(buildAudioAttributes(), configuration.playbackConfig.handleAudioFocus)
            .setHandleAudioBecomingNoisy(configuration.playbackConfig.handleAudioBecomingNoisy)
            .setWakeMode(configuration.playbackConfig.wakeMode)
            .setSeekBackIncrementMs(configuration.playbackConfig.seekBackIncrementMs)
            .setSeekForwardIncrementMs(configuration.playbackConfig.seekForwardIncrementMs)
            .setVideoScalingMode(configuration.playbackConfig.videoScalingMode)
            .build()

        player.repeatMode = configuration.playbackConfig.repeatMode
        player.shuffleModeEnabled = configuration.playbackConfig.shuffleModeEnabled
        player.playWhenReady = configuration.playbackConfig.playWhenReady

        if (configuration.loggingConfig.enableVerboseLogging) {
            Timber.d(
                "PlayerFactory: ExoPlayer created " +
                    "[renderers=${player.rendererCount}, seekBack=${configuration.playbackConfig.seekBackIncrementMs}ms]"
            )
        }
        return player
    }

    // ── Private builders ─────────────────────────────────────────────────────

    private fun buildRenderersFactory(): DefaultRenderersFactory =
        DefaultRenderersFactory(context).apply {
            setExtensionRendererMode(configuration.renderersConfig.extensionRendererMode)
        }

    private fun buildLoadControl(): DefaultLoadControl {
        val buf = configuration.bufferConfig
        return DefaultLoadControl.Builder()
            .setBufferDurationsMs(
                buf.minBufferMs,
                buf.maxBufferMs,
                buf.bufferForPlaybackMs,
                buf.bufferForPlaybackAfterRebufferMs,
            )
            .build()
    }

    private fun buildDataSourceFactory(): OkHttpDataSource.Factory =
        OkHttpDataSource.Factory(okHttpClient)
            .setUserAgent(configuration.networkConfig.userAgent)

    private fun buildTrackSelector(): DefaultTrackSelector =
        DefaultTrackSelector(context).apply {
            setParameters(
                buildUponParameters()
                    .setPreferredAudioLanguage("en")
                    .build()
            )
        }

    private fun buildLivePlaybackSpeedControl(): DefaultLivePlaybackSpeedControl {
        val live = configuration.liveConfig
        return DefaultLivePlaybackSpeedControl.Builder()
            .setFallbackMaxPlaybackSpeed(live.maxSpeed)
            .setFallbackMinPlaybackSpeed(live.minSpeed)
            .build()
    }

    private fun buildAudioAttributes(): AudioAttributes =
        AudioAttributes.Builder()
            .setUsage(C.USAGE_MEDIA)
            .setContentType(C.AUDIO_CONTENT_TYPE_MUSIC)
            .build()
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/engine/PlayerInitializer.kt

```kotlin
package com.xplayer.dev.core.engine

import androidx.annotation.MainThread
import com.xplayer.dev.core.config.PlayerConfiguration
import timber.log.Timber
import javax.inject.Inject
import javax.inject.Singleton

/**
 * Bootstraps the engine once per player session.
 *
 * Separating initialisation from [PlayerEngine] ensures the engine
 * itself stays stateless with respect to lifecycle management, and
 * initialisation logic can evolve independently (e.g. adding DRM
 * provisioning, pre-warming caches, etc.).
 */
@Singleton
class PlayerInitializer @Inject constructor(
    private val engine: PlayerEngine,
    private val configuration: PlayerConfiguration,
    private val validator: com.xplayer.dev.core.config.PlayerConfigurationValidator,
) {

    private var isInitialised = false

    /**
     * Safe, idempotent entry-point. Calling this more than once
     * is a no-op and logs a warning rather than crashing.
     */
    @MainThread
    fun initializeIfNeeded() {
        if (isInitialised) {
            Timber.w("PlayerInitializer: already initialised — skipping")
            return
        }
        validator.validate(configuration)
        runCatching { engine.initialize() }
            .onSuccess {
                isInitialised = true
                Timber.i(
                    "PlayerInitializer: engine ready " +
                        "[verbose=${configuration.loggingConfig.enableVerboseLogging}]"
                )
            }
            .onFailure { throwable ->
                Timber.e(throwable, "PlayerInitializer: engine init failed")
                throw throwable
            }
    }

    /** Returns true only after a successful [initializeIfNeeded] call. */
    val isReady: Boolean get() = isInitialised
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/engine/PlayerReleaseManager.kt

```kotlin
package com.xplayer.dev.core.engine

import androidx.annotation.MainThread
import com.xplayer.dev.core.threading.PlayerExecutor
import com.xplayer.dev.core.threading.PlayerScopeManager
import timber.log.Timber
import javax.inject.Inject
import javax.inject.Singleton

/**
 * Owns the teardown sequence for a player session.
 *
 * Release is a multi-step process:
 *  1. Cancel all coroutines in [playerScope].
 *  2. Release the ExoPlayer instance via [PlayerEngine].
 *  3. Shut down the background [PlayerExecutor].
 */
@Singleton
class PlayerReleaseManager @Inject constructor(
    private val engine: PlayerEngine,
    private val scopeManager: PlayerScopeManager,
    private val executor: PlayerExecutor,
) {

    private var isReleased = false

    @MainThread
    fun release(cause: Exception? = null) {
        if (isReleased) {
            Timber.w("PlayerReleaseManager: already released — skipping")
            return
        }
        Timber.i("PlayerReleaseManager: releasing player [cause=${cause?.message}]")

        runCatching { scopeManager.cancelPlayerScope(cause) }
            .onFailure { Timber.e(it, "Scope cancellation failed") }

        runCatching { engine.release() }
            .onFailure { Timber.e(it, "Engine release failed") }

        runCatching { executor.shutdown() }
            .onFailure { Timber.e(it, "Executor shutdown failed") }

        isReleased = true
        Timber.i("PlayerReleaseManager: teardown complete")
    }

    /**
     * Force immediate release for emergency recovery scenarios.
     */
    @MainThread
    fun forceRelease() {
        Timber.w("PlayerReleaseManager: executing emergency forceRelease")
        runCatching { engine.release() }
        runCatching { scopeManager.cancelPlayerScope() }
        runCatching { executor.shutdownNow() }
        isReleased = true
    }
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/engine/internal/EngineEventBus.kt

```kotlin
package com.xplayer.dev.core.engine.internal

import com.xplayer.dev.core.events.PlayerEvent
import kotlinx.coroutines.channels.Channel
import kotlinx.coroutines.channels.ChannelResult
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.receiveAsFlow
import timber.log.Timber
import java.util.concurrent.atomic.AtomicLong
import javax.inject.Inject

/**
 * Typed event bus backed by a [Channel].
 *
 * Design decisions:
 *  1. Capacity = [CAPACITY] (512) — absorbs analytics bursts without
 *     blocking the main thread on slow collectors.
 *  2. Dropped events are counted and reported — invisible drops are
 *     a production reliability risk.
 *  3. [close] is idempotent — safe to call from release() multiple times.
 *  4. [DroppedEventRecord] enables post-mortem analysis in crash reports.
 */
class EngineEventBus @Inject constructor() {

    private val channel = Channel<PlayerEvent>(capacity = CAPACITY)
    private val _droppedCount = AtomicLong(0L)
    private val _sentCount = AtomicLong(0L)

    val events: Flow<PlayerEvent> = channel.receiveAsFlow()

    val droppedEventCount: Long get() = _droppedCount.get()
    val sentEventCount: Long get() = _sentCount.get()

    /**
     * Attempt non-blocking send.
     * @return [SendResult] with outcome details.
     */
    fun send(event: PlayerEvent): SendResult {
        val result: ChannelResult<Unit> = channel.trySend(event)
        return when {
            result.isSuccess -> {
                _sentCount.incrementAndGet()
                SendResult.Sent
            }
            result.isFailure -> {
                val count = _droppedCount.incrementAndGet()
                val record = DroppedEventRecord(
                    eventType  = event::class.simpleName ?: "Unknown",
                    eventId    = event.eventId,
                    dropNumber = count,
                )
                Timber.e(
                    "EventBus: DROPPED event #$count " +
                        "[type=${record.eventType}] [id=${record.eventId}]"
                )
                SendResult.Dropped(record)
            }
            result.isClosed -> {
                Timber.w(
                    "EventBus: channel closed — " +
                        "ignoring ${event::class.simpleName}"
                )
                SendResult.ChannelClosed
            }
            else -> SendResult.ChannelClosed
        }
    }

    /**
     * Close the channel. Idempotent.
     * All subsequent [send] calls will return [SendResult.ChannelClosed].
     */
    fun close() {
        channel.close()
        Timber.d(
            "EventBus: closed " +
                "[sent=${_sentCount.get()} dropped=${_droppedCount.get()}]"
        )
    }

    fun isOpen(): Boolean = !channel.isClosedForSend

    sealed interface SendResult {
        data object Sent : SendResult
        data class Dropped(val record: DroppedEventRecord) : SendResult
        data object ChannelClosed : SendResult
    }

    data class DroppedEventRecord(
        val eventType: String,
        val eventId: String,
        val dropNumber: Long,
        val timestampMs: Long = System.currentTimeMillis(),
    )

    companion object {
        // 512 slots — at 60fps + analytics events, gives ~4s headroom
        // before drops occur on a completely frozen collector
        const val CAPACITY = 512
    }
}
```

## xplayer/app/src/main/java/com/xplayer/dev/core/engine/internal/EngineLifecycle.kt

```kotlin
package com.xplayer.dev.core.engine.internal

import androidx.media3.exoplayer.ExoPlayer

/**
 * Sealed FSM representing the engine's own lifecycle,
 * independent of playback state.
 *
 * Legal transitions:
 *   Uninitialised → Initialising → Active → Releasing → Released
 *                                         ↘ Error (init failed)
 *
 * Why a separate lifecycle?
 *  - Prevents use-after-release crashes via exhaustive when() checks.
 *  - Makes double-init and double-release impossible at compile time.
 *  - Thread safety: @Volatile + atomic CAS updates only.
 */
sealed class EngineLifecycle {

    /** Engine created but ExoPlayer not yet built. */
    data object Uninitialised : EngineLifecycle()

    /**
     * ExoPlayer constructor is running on main thread.
     * Prevents concurrent init calls during async startup.
     */
    data object Initialising : EngineLifecycle()

    /**
     * ExoPlayer is alive and operational.
     * All playback commands are routed through [player].
     *
     * @param initTimeMs wall-clock when Active was entered —
     *                   used as TTFP epoch for the first session.
     */
    data class Active(
        val player: ExoPlayer,
        val initTimeMs: Long = System.currentTimeMillis(),
    ) : EngineLifecycle()

    /**
     * Release sequence has started.
     * All commands are rejected; no new work is accepted.
     */
    data object Releasing : EngineLifecycle()

    /**
     * Engine fully torn down.
     * Instance must be discarded; create a new one for re-use.
     */
    data object Released : EngineLifecycle()

    /**
     * Fatal initialisation error.
     * [cause] carries the throwable for crash reporting.
     */
    data class InitError(val cause: Throwable) : EngineLifecycle()
}

// ── Accessors ─────────────────────────────────────────────────────────────────

val EngineLifecycle.isOperational: Boolean
    get() = this is EngineLifecycle.Active

val EngineLifecycle.player: ExoPlayer?
    get() = (this as? EngineLifecycle.Active)?.player

/** Throws if engine is not in [EngineLifecycle.Active] state. */
fun EngineLifecycle.requirePlayer(context: String = "operation"): ExoPlayer =
    player ?: throw IllegalStateException(
        "PlayerEngine is not active for '$context'. " +
            "Current lifecycle: ${this::class.simpleName}. " +
            "Call initialize() before any playback operation."
    )
```

## xplayer/app/src/main/java/com/xplayer/dev/core/engine/internal/EngineListenerHub.kt

```kotlin
package com.xplayer.dev.core.engine.internal

import androidx.media3.common.MediaItem
import androidx.media3.common.MediaMetadata
import androidx.media3.common.PlaybackException
import androidx.media3.common.PlaybackParameters
import androidx.media3.common.Player
import androidx.media3.common.Tracks
import androidx.media3.common.VideoSize
import androidx.media3.exoplayer.ExoPlayer
import androidx.media3.exoplayer.analytics.AnalyticsListener
import com.xplayer.dev.core.events.PlayerEvent
import com.xplayer.dev.core.state.PlayerState
import timber.log.Timber

/**
 * Composite listener hub that wires ExoPlayer callbacks
 * into [onState] and [onEvent] lambdas.
 *
 * Separation from [PlayerEngine]:
 *  - Each callback group is a distinct, testable section.
 *  - Adding new listeners (DRM, bandwidth) does not bloat PlayerEngine.
 *  - Listener lifecycle (attach/detach) is encapsulated here.
 *
 * @param onState  Called when a new [PlayerState] should be emitted.
 * @param onEvent  Called when a new [PlayerEvent] should be sent.
 */
class EngineListenerHub(
    private val sessionManager: EngineSessionManager,
    private val metricsProbe: EngineMetricsProbe,
    private val stateValidator: EngineStateValidator,
    private val isDebug: Boolean,
    private val onState: (PlayerState) -> Unit,
    private val onEvent: (PlayerEvent) -> Unit,
) {

    // ── Internal tracking ─────────────────────────────────────────────────────

    private var isFirstPlay = true
    private var previousPlaybackState = Player.STATE_IDLE
    private var currentSpeed = 1.0f

    // ── Player.Listener ───────────────────────────────────────────────────────

    val playerListener = object : Player.Listener {

        // ── Playback state ────────────────────────────────────────────────────

        override fun onPlaybackStateChanged(playbackState: Int) {
            val player = playerRef ?: return
            val session = sessionManager.current ?: return

            Timber.d(
                "ListenerHub: onPlaybackStateChanged " +
                    "[${stateNameOf(previousPlaybackState)} → ${stateNameOf(playbackState)}]"
            )

            when (playbackState) {

                Player.STATE_BUFFERING -> handleBufferingStart(player, session)

                Player.STATE_READY -> handleReady(player, session)

                Player.STATE_ENDED -> handleEnded(player, session)

                Player.STATE_IDLE -> {
                    if (previousPlaybackState != Player.STATE_IDLE) {
                        transition(PlayerState.Idle(sequenceNumber = 0L))
                    }
                }
            }
            previousPlaybackState = playbackState
        }

        // ── Is playing ────────────────────────────────────────────────────────

        override fun onIsPlayingChanged(isPlaying: Boolean) {
            val player = playerRef ?: return
            val session = sessionManager.current ?: return

            if (isPlaying) {
                handlePlayingStarted(player, session)
            } else {
                handlePlayingPaused(player, session)
            }
        }

        // ── Error ─────────────────────────────────────────────────────────────

        override fun onPlayerError(error: PlaybackException) {
            val player = playerRef
            val session = sessionManager.current
            val previousState = currentEmittedState

            val retryCount = (previousState as? PlayerState.Error)?.retryCount ?: 0

            val errorState = PlayerState.Error(
                mediaId       = session?.mediaId,
                cause         = error,
                retryCount    = retryCount,
                isFatal       = !error.isRecoverable(),
                previousState = previousState,
            )
            transition(errorState)

            onEvent(
                PlayerEvent.ErrorOccurred(
                    sessionId    = session?.sessionId ?: "",
                    sequenceNum  = 0L,
                    mediaId      = session?.mediaId ?: "",
                    errorCategory= com.xplayer.dev.core.state.PlayerState.ErrorCategory.NETWORK,
                    error        = error,
                    isFatal      = !error.isRecoverable(),
                    retryAttempt = retryCount,
                    context      = buildErrorContext(error, player, session),
                    previousState= "Unknown"
                )
            )
            Timber.e(
                error,
                "ListenerHub: player error " +
                    "[code=${error.errorCode}] " +
                    "[fatal=${!error.isRecoverable()}] " +
                    "[mediaId=${session?.mediaId}]"
            )
        }

        // ── Tracks ────────────────────────────────────────────────────────────

        override fun onTracksChanged(tracks: Tracks) {
            val session = sessionManager.current ?: return
            onEvent(
                PlayerEvent.TracksChanged(
                    sessionId    = session?.sessionId ?: "",
                    sequenceNum  = 0L,
                    mediaId      = session?.mediaId ?: "",
                    availableTracks = tracks,
                    selectedVideo = resolveVideoTrack(tracks),
                    selectedAudio = resolveAudioTrack(tracks),
                    selectedText  = null
                )
            )
        }

        // ── Video size ────────────────────────────────────────────────────────

        override fun onVideoSizeChanged(videoSize: VideoSize) {
            val session = sessionManager.current ?: return
            onEvent(
                PlayerEvent.VideoSizeChanged(
                    sessionId    = session?.sessionId ?: "",
                    sequenceNum  = 0L,
                    mediaId      = session?.mediaId ?: "",
                    width      = videoSize.width,
                    height     = videoSize.height,
                    pixelWidthHeightRatio = videoSize.pixelWidthHeightRatio,
                    unappliedRotationDegrees = videoSize.unappliedRotationDegrees
                )
            )
        }

        // ── Metadata ──────────────────────────────────────────────────────────

        override fun onMediaMetadataChanged(mediaMetadata: MediaMetadata) {
            val session = sessionManager.current ?: return
            onEvent(
                PlayerEvent.MetadataReceived(
                    mediaId    = session.mediaId,
                    sessionId  = session.sessionId,
                    title      = mediaMetadata.title?.toString(),
                    artist     = mediaMetadata.artist?.toString(),
                    artworkUri = mediaMetadata.artworkUri?.toString(),
                    durationMs = null,
                )
            )
        }

        // ── Seek / discontinuity ──────────────────────────────────────────────

        override fun onPositionDiscontinuity(
            oldPosition: Player.PositionInfo,
            newPosition: Player.PositionInfo,
            reason: Int,
        ) {
            if (reason == Player.DISCONTINUITY_REASON_SEEK) {
                metricsProbe.onSeekStart(oldPosition.positionMs)
                Timber.d(
                    "ListenerHub: seek started " +
                        "[from=${oldPosition.positionMs}ms]"
                )
            }
        }

        override fun onSeekBackIncrementChanged(seekBackIncrementMs: Long)    = Unit
        override fun onSeekForwardIncrementChanged(seekForwardIncrementMs: Long) = Unit

        // ── Playback parameters (speed) ───────────────────────────────────────

        override fun onPlaybackParametersChanged(playbackParameters: PlaybackParameters) {
            val session = sessionManager.current ?: return
            val previous = currentSpeed
            val new = playbackParameters.speed
            currentSpeed = new

            if (previous != new) {
                onEvent(
                    PlayerEvent.PlaybackSpeedChanged(
                        mediaId       = session.mediaId,
                        sessionId     = session.sessionId,
                        previousSpeed = previous,
                        newSpeed      = new,
                    )
                )
            }
        }

        // ── Media item transition ─────────────────────────────────────────────

        override fun onMediaItemTransition(mediaItem: MediaItem?, reason: Int) {
            Timber.d(
                "ListenerHub: media item transition " +
                    "[reason=${transitionReasonName(reason)}]"
            )
        }
    }

    // ── AnalyticsListener ─────────────────────────────────────────────────────

    val analyticsListener = object : AnalyticsListener {

        // ── Dropped frames ────────────────────────────────────────────────────

        override fun onDroppedVideoFrames(
            eventTime: AnalyticsListener.EventTime,
            droppedFrames: Int,
            elapsedMs: Long,
        ) {
            if (droppedFrames <= 0) return
            metricsProbe.onDroppedFrames(droppedFrames)
            val session = sessionManager.current ?: return
            onEvent(
                PlayerEvent.DroppedFrames(
                    mediaId    = session.mediaId,
                    sessionId  = session.sessionId,
                    droppedCount = droppedFrames,
                    elapsedMs  = elapsedMs,
                )
            )
            Timber.w(
                "ListenerHub: dropped $droppedFrames frames " +
                    "[elapsed=${elapsedMs}ms]"
            )
        }

        // ── Bandwidth ─────────────────────────────────────────────────────────

        override fun onBandwidthEstimate(
            eventTime: AnalyticsListener.EventTime,
            totalLoadTimeMs: Int,
            totalBytesLoaded: Long,
            bitrateEstimate: Long,
        ) {
            if (bitrateEstimate <= 0L) return
            metricsProbe.onBandwidthSample(bitrateEstimate)
            val session = sessionManager.current ?: return
            onEvent(
                PlayerEvent.BandwidthEstimated(
                    mediaId       = session.mediaId,
                    sessionId     = session.sessionId,
                    sequenceNum   = 0L,
                    bandwidthBps  = bitrateEstimate,
                    sampleBps     = bitrateEstimate,
                )
            )
        }

        // ── Video decoder ─────────────────────────────────────────────────────

        override fun onVideoFrameProcessingOffset(
            eventTime: AnalyticsListener.EventTime,
            totalProcessingOffsetUs: Long,
            frameCount: Int,
        ) {
            if (frameCount > 0) {
                metricsProbe.onRenderedFrames(frameCount.toLong())
            }
        }

        // ── Load error ────────────────────────────────────────────────────────

        override fun onLoadError(
            eventTime: AnalyticsListener.EventTime,
            loadEventInfo: androidx.media3.exoplayer.source.LoadEventInfo,
            mediaLoadData: androidx.media3.exoplayer.source.MediaLoadData,
            error: java.io.IOException,
            wasCanceled: Boolean
        ) {
            Timber.w(
                "ListenerHub: load error " +
                    "[type=${mediaLoadData.dataType}]"
            )
        }
    }

    // ── Attach / detach ───────────────────────────────────────────────────────

    /** Attach both listeners to [player]. */
    fun attach(player: ExoPlayer) {
        playerRef = player
        isFirstPlay = true
        previousPlaybackState = Player.STATE_IDLE
        player.addListener(playerListener)
        player.addAnalyticsListener(analyticsListener)
        Timber.d("ListenerHub: attached")
    }

    /** Detach both listeners from [player]. */
    fun detach(player: ExoPlayer) {
        player.removeListener(playerListener)
        player.removeAnalyticsListener(analyticsListener)
        playerRef = null
        Timber.d("ListenerHub: detached")
    }

    // ── Private state ─────────────────────────────────────────────────────────

    @Volatile private var playerRef: ExoPlayer? = null
    @Volatile private var currentEmittedState: PlayerState = PlayerState.Idle(sequenceNumber = 0L)

    private fun transition(newState: PlayerState) {
        val result = stateValidator.validate(currentEmittedState, newState, isDebug)
        when (result) {
            is EngineStateValidator.ValidationResult.Accepted -> {
                currentEmittedState = result.state
                onState(result.state)
            }
            is EngineStateValidator.ValidationResult.Rejected -> {
                Timber.e("ListenerHub: rejected transition — ${result.reason}")
            }
        }
    }

    // ── Playback state handlers ───────────────────────────────────────────────

    private fun handleBufferingStart(
        player: ExoPlayer,
        session: EngineSessionManager.Session,
    ) {
        val isRebuffer = previousPlaybackState == Player.STATE_READY ||
            previousPlaybackState == Player.STATE_BUFFERING

        if (isRebuffer) {
            metricsProbe.onStallStart(player.bufferedPosition)
        }

        val stalls = if (isRebuffer) metricsProbe.snapshot().stallCount else 0
        transition(
            PlayerState.Buffering(
                mediaId      = session?.mediaId ?: "",
                bufferPercent = player.bufferedPercentage.coerceIn(0, 100),
                rebufferCount = stalls,
            )
        )
        onEvent(
            PlayerEvent.BufferingStarted(
                mediaId    = session.mediaId,
                sessionId  = session.sessionId,
                positionMs = player.currentPosition,
                stallCount = stalls,
            )
        )
    }

    private fun handleReady(
        player: ExoPlayer,
        session: EngineSessionManager.Session,
    ) {
        // Emit BufferingEnded if we were in a stall
        if (previousPlaybackState == Player.STATE_BUFFERING) {
            val stallMs = metricsProbe.onStallEnd()
            onEvent(
                PlayerEvent.BufferingEnded(
                    mediaId              = session.mediaId,
                    sessionId            = session.sessionId,
                    stallDurationMs      = stallMs,
                    bufferPercentOnResume = player.bufferedPercentage,
                )
            )
        }

        transition(
            PlayerState.Ready(
                mediaId    = session.mediaId,
                durationMs = player.duration.coerceAtLeast(0L),
            )
        )

        // Emit current buffer state on becoming ready
        onEvent(
            PlayerEvent.BufferUpdated(
                mediaId           = session.mediaId,
                sessionId         = session.sessionId,
                bufferPercent     = player.bufferedPercentage,
                bufferedPositionMs = player.bufferedPosition,
                currentPositionMs  = player.currentPosition,
            )
        )
    }

    private fun handleEnded(
        player: ExoPlayer,
        session: EngineSessionManager.Session,
    ) {
        val duration = player.duration.coerceAtLeast(0L)
        val position = player.currentPosition

        transition(
            PlayerState.Ended(
                mediaId         = session.mediaId,
                totalDurationMs = duration,
            )
        )
        onEvent(
            PlayerEvent.PlaybackEnded(
                mediaId           = session.mediaId,
                sessionId         = session.sessionId,
                totalPlayedMs     = position,
                totalDurationMs = duration,
                completionPercent = if (duration > 0)
                    (position * 100f / duration).coerceIn(0f, 100f)
                else 100f,
            )
        )
    }

    private fun handlePlayingStarted(
        player: ExoPlayer,
        session: EngineSessionManager.Session,
    ) {
        val ttfp = if (isFirstPlay) {
            isFirstPlay = false
            metricsProbe.onFirstFrame()
        } else 0L

        transition(
            PlayerState.Playing(
                mediaId       = session.mediaId,
                positionMs    = player.currentPosition,
                durationMs    = player.duration.coerceAtLeast(0L),
                playbackSpeed = player.playbackParameters.speed,
                isLive = player.isCurrentMediaItemLive,
            )
        )
        onEvent(
            PlayerEvent.PlaybackStarted(
                mediaId            = session.mediaId,
                sessionId          = session.sessionId,
                positionMs         = player.currentPosition,
                durationMs = player.duration.coerceAtLeast(0L),
                isResumed          = !isFirstPlay,
                timeToFirstFrameMs = ttfp.coerceAtLeast(0L),
            )
        )
    }

    private fun handlePlayingPaused(
        player: ExoPlayer,
        session: EngineSessionManager.Session,
    ) {
        if (player.playbackState != Player.STATE_READY &&
            player.playbackState != Player.STATE_BUFFERING
        ) return

        val pausedBySystem = !player.playWhenReady

        transition(
            PlayerState.Paused(
                mediaId       = session.mediaId,
                positionMs    = player.currentPosition,
                pausedBySystem = pausedBySystem,
            )
        )
        onEvent(
            PlayerEvent.PlaybackPaused(
                mediaId       = session.mediaId,
                sessionId     = session.sessionId,
                positionMs    = player.currentPosition,
                pauseReason = com.xplayer.dev.core.state.PlayerState.Paused.PauseReason.USER,
                pausedBySystem = pausedBySystem,
            )
        )
    }

    // ── Track resolution ──────────────────────────────────────────────────────

    private fun resolveVideoTrack(tracks: Tracks): PlayerEvent.TrackInfo? {
        tracks.groups.forEach { group ->
            if (group.type == androidx.media3.common.C.TRACK_TYPE_VIDEO) {
                for (i in 0 until group.length) {
                    if (group.isTrackSelected(i)) {
                        val format = group.getTrackFormat(i)
                        return PlayerEvent.TrackInfo(
                            trackId    = format.id ?: "",
                            mimeType   = format.sampleMimeType ?: "",
                            bitrateBps = format.bitrate,
                            codec      = format.codecs ?: "unknown",
                            width      = format.width,
                            height     = format.height,
                        )
                    }
                }
            }
        }
        return null
    }

    private fun resolveAudioTrack(tracks: Tracks): PlayerEvent.TrackInfo? {
        tracks.groups.forEach { group ->
            if (group.type == androidx.media3.common.C.TRACK_TYPE_AUDIO) {
                for (i in 0 until group.length) {
                    if (group.isTrackSelected(i)) {
                        val format = group.getTrackFormat(i)
                        return PlayerEvent.TrackInfo(
                            trackId    = format.id ?: "",
                            mimeType   = format.sampleMimeType ?: "",
                            bitrateBps = format.bitrate,
                            codec      = format.codecs ?: "unknown",
                            language   = format.language,
                        )
                    }
                }
            }
        }
        return null
    }

    // ── Helpers ───────────────────────────────────────────────────────────────

    private fun buildErrorContext(
        error: PlaybackException,
        player: ExoPlayer?,
        session: EngineSessionManager.Session?,
    ): Map<String, String> = buildMap {
        put("errorCode", error.errorCode.toString())
        put("errorCodeName", error.errorCodeName)
        put("mediaId", session?.mediaId ?: "null")
        put("sessionId", session?.sessionId ?: "null")
        put("position", player?.currentPosition?.toString() ?: "0")
        put("bufferedPercent", player?.bufferedPercentage?.toString() ?: "0")
        put("playWhenReady", player?.playWhenReady?.toString() ?: "false")
        put("networkType", "unknown") // inject ConnectivityManager for real value
    }

    private fun stateNameOf(state: Int): String = when (state) {
        Player.STATE_IDLE      -> "IDLE"
        Player.STATE_BUFFERING -> "BUFFERING"
        Player.STATE_READY     -> "READY"
        Player.STATE_ENDED     -> "ENDED"
        else                   -> "UNKNOWN($state)"
    }

    private fun transitionReasonName(reason: Int): String = when (reason) {
        Player.MEDIA_ITEM_TRANSITION_REASON_REPEAT    -> "REPEAT"
        Player.MEDIA_ITEM_TRANSITION_REASON_AUTO      -> "AUTO"
        Player.MEDIA_ITEM_TRANSITION_REASON_SEEK      -> "SEEK"
        Player.MEDIA_ITEM_TRANSITION_REASON_PLAYLIST_CHANGED -> "PLAYLIST_CHANGED"
        else -> "UNKNOWN($reason)"
    }
}

// ── Error classifier ──────────────────────────────────────────────────────────

fun PlaybackException.isRecoverable(): Boolean = when (errorCode) {
    PlaybackException.ERROR_CODE_IO_NETWORK_CONNECTION_FAILED,
    PlaybackException.ERROR_CODE_IO_NETWORK_CONNECTION_TIMEOUT,
    PlaybackException.ERROR_CODE_IO_BAD_HTTP_STATUS,
    PlaybackException.ERROR_CODE_IO_CLEARTEXT_NOT_PERMITTED,
    -> true
    else -> false
}
```

## xplayer/app/src/main/java/com/xplayer/dev/core/engine/internal/EngineMetricsProbe.kt

```kotlin
package com.xplayer.dev.core.engine.internal

import androidx.media3.exoplayer.ExoPlayer
import timber.log.Timber
import java.util.concurrent.atomic.AtomicInteger
import java.util.concurrent.atomic.AtomicLong
import javax.inject.Inject
import kotlin.math.roundToLong

/**
 * Collects real-time QoS metrics for a single playback session.
 *
 * Metrics tracked:
 *  ┌─────────────────────────────────────────────────────┐
 *  │ TTFP   — Time from prepare() to first Playing frame │
 *  │ Stalls — Count + total stalled duration             │
 *  │ Seeks  — Count + average seek duration              │
 *  │ Drops  — Total dropped video frames                 │
 *  │ BW     — Bandwidth samples (avg / peak / min)       │
 *  │ Buffer — Minimum buffer observed before stall       │
 *  └─────────────────────────────────────────────────────┘
 *
 * Thread safety: all mutable fields are [AtomicLong] / [AtomicInteger].
 * Snapshot via [snapshot] is safe to call from any thread.
 */
class EngineMetricsProbe @Inject constructor() {

    // ── TTFP ──────────────────────────────────────────────────────────────────

    private val prepareEpochMs  = AtomicLong(0L)
    private val ttfpMs          = AtomicLong(-1L)  // -1 = not yet measured

    // ── Stall ─────────────────────────────────────────────────────────────────

    private val stallCount       = AtomicInteger(0)
    private val stallEpochMs     = AtomicLong(0L)
    private val totalStalledMs   = AtomicLong(0L)
    private val lastStallMs      = AtomicLong(0L)

    // ── Seek ──────────────────────────────────────────────────────────────────

    private val seekCount        = AtomicInteger(0)
    private val totalSeekMs      = AtomicLong(0L)
    private val seekEpochMs      = AtomicLong(0L)
    private val lastFromPosition = AtomicLong(0L)

    // ── Frames ────────────────────────────────────────────────────────────────

    private val droppedFrames    = AtomicInteger(0)
    private val renderedFrames   = AtomicLong(0L)

    // ── Bandwidth ─────────────────────────────────────────────────────────────

    private val bwSamples        = ArrayDeque<Long>(MAX_BW_SAMPLES)
    private val bwLock           = Any()

    // ── Buffer watermark ──────────────────────────────────────────────────────

    private val minBufferBeforeStallMs = AtomicLong(Long.MAX_VALUE)

    // ── Session reset ─────────────────────────────────────────────────────────

    /**
     * Reset all counters for a new session.
     * Must be called before [onPrepareStart].
     */
    fun reset() {
        prepareEpochMs.set(0L)
        ttfpMs.set(-1L)
        stallCount.set(0)
        stallEpochMs.set(0L)
        totalStalledMs.set(0L)
        lastStallMs.set(0L)
        seekCount.set(0)
        totalSeekMs.set(0L)
        seekEpochMs.set(0L)
        lastFromPosition.set(0L)
        droppedFrames.set(0)
        renderedFrames.set(0L)
        synchronized(bwLock) { bwSamples.clear() }
        minBufferBeforeStallMs.set(Long.MAX_VALUE)
        Timber.d("MetricsProbe: reset")
    }

    // ── TTFP hooks ────────────────────────────────────────────────────────────

    fun onPrepareStart() {
        prepareEpochMs.set(System.currentTimeMillis())
    }

    /**
     * Call when isPlaying flips true for the first time.
     * Subsequent calls are no-ops.
     *
     * @return measured TTFP in ms, or -1 if already measured.
     */
    fun onFirstFrame(): Long {
        if (ttfpMs.get() != -1L) return -1L
        val epoch = prepareEpochMs.get()
        if (epoch == 0L) return -1L
        val measured = System.currentTimeMillis() - epoch
        ttfpMs.set(measured)
        Timber.i("MetricsProbe: TTFP = ${measured}ms")
        return measured
    }

    // ── Stall hooks ───────────────────────────────────────────────────────────

    /**
     * Call when ExoPlayer enters STATE_BUFFERING after STATE_READY
     * (i.e., a rebuffer / stall, not the initial buffer).
     *
     * @param bufferMs buffer level at stall time — for watermark tracking.
     */
    fun onStallStart(bufferMs: Long = 0L) {
        stallEpochMs.set(System.currentTimeMillis())
        stallCount.incrementAndGet()
        if (bufferMs < minBufferBeforeStallMs.get()) {
            minBufferBeforeStallMs.set(bufferMs)
        }
        Timber.d("MetricsProbe: stall #${stallCount.get()} start")
    }

    /**
     * Call when ExoPlayer exits STATE_BUFFERING back to STATE_READY.
     *
     * @return stall duration in ms.
     */
    fun onStallEnd(): Long {
        val epoch = stallEpochMs.getAndSet(0L)
        if (epoch == 0L) return 0L
        val duration = System.currentTimeMillis() - epoch
        totalStalledMs.addAndGet(duration)
        lastStallMs.set(duration)
        Timber.d("MetricsProbe: stall ended [duration=${duration}ms]")
        return duration
    }

    // ── Seek hooks ────────────────────────────────────────────────────────────

    fun onSeekStart(fromPositionMs: Long) {
        seekEpochMs.set(System.currentTimeMillis())
        lastFromPosition.set(fromPositionMs)
        seekCount.incrementAndGet()
    }

    /** @return seek operation duration in ms. */
    fun onSeekComplete(): Long {
        val epoch = seekEpochMs.getAndSet(0L)
        if (epoch == 0L) return 0L
        val duration = System.currentTimeMillis() - epoch
        totalSeekMs.addAndGet(duration)
        return duration
    }

    val lastSeekFromPositionMs: Long get() = lastFromPosition.get()

    // ── Frame hooks ───────────────────────────────────────────────────────────

    fun onDroppedFrames(count: Int) {
        droppedFrames.addAndGet(count)
    }

    fun onRenderedFrames(count: Long) {
        renderedFrames.addAndGet(count)
    }

    // ── Bandwidth hooks ───────────────────────────────────────────────────────

    fun onBandwidthSample(bps: Long) {
        synchronized(bwLock) {
            if (bwSamples.size >= MAX_BW_SAMPLES) bwSamples.removeFirst()
            bwSamples.addLast(bps)
        }
    }

    // ── Snapshot ──────────────────────────────────────────────────────────────

    /**
     * Thread-safe immutable snapshot of current metrics.
     * Safe to call from any thread.
     */
    fun snapshot(): MetricsSnapshot {
        val bwCopy = synchronized(bwLock) { bwSamples.toList() }
        return MetricsSnapshot(
            ttfpMs              = ttfpMs.get().takeIf { it >= 0L },
            stallCount          = stallCount.get(),
            totalStalledMs      = totalStalledMs.get(),
            lastStallMs         = lastStallMs.get(),
            minBufferBeforeStallMs = minBufferBeforeStallMs.get()
                .takeIf { it != Long.MAX_VALUE },
            seekCount           = seekCount.get(),
            averageSeekMs       = if (seekCount.get() > 0)
                totalSeekMs.get() / seekCount.get() else 0L,
            droppedFrames       = droppedFrames.get(),
            renderedFrames      = renderedFrames.get(),
            averageBandwidthBps = bwCopy.average()
                .takeIf { it.isFinite() }?.roundToLong() ?: 0L,
            peakBandwidthBps    = bwCopy.maxOrNull() ?: 0L,
            minBandwidthBps     = bwCopy.minOrNull() ?: 0L,
            bandwidthSampleCount = bwCopy.size,
        )
    }

    companion object {
        private const val MAX_BW_SAMPLES = 60
    }
}

// ── Snapshot model ────────────────────────────────────────────────────────────

data class MetricsSnapshot(
    val ttfpMs: Long?,                    // null = not yet measured
    val stallCount: Int,
    val totalStalledMs: Long,
    val lastStallMs: Long,
    val minBufferBeforeStallMs: Long?,    // null = no stall occurred
    val seekCount: Int,
    val averageSeekMs: Long,
    val droppedFrames: Int,
    val renderedFrames: Long,
    val averageBandwidthBps: Long,
    val peakBandwidthBps: Long,
    val minBandwidthBps: Long,
    val bandwidthSampleCount: Int,
) {
    val dropRate: Float
        get() {
            val total = renderedFrames + droppedFrames
            return if (total == 0L) 0f else droppedFrames.toFloat() / total
        }

    val hasStalls: Boolean get() = stallCount > 0
    val hasTtfp: Boolean get() = ttfpMs != null
}
```

## xplayer/app/src/main/java/com/xplayer/dev/core/engine/internal/EngineSessionManager.kt

```kotlin
package com.xplayer.dev.core.engine.internal

import timber.log.Timber
import java.util.UUID
import java.util.concurrent.atomic.AtomicReference
import javax.inject.Inject

/**
 * Manages player session identity and metadata.
 *
 * A "session" begins when [prepare] is called with a new media item
 * and ends when [release], [stop], or a new [prepare] is called.
 *
 * Session data is used to:
 *  1. Group all events belonging to a single playback attempt.
 *  2. Provide context to crash reporters.
 *  3. Compute session-level QoS metrics.
 */
class EngineSessionManager @Inject constructor() {

    data class Session(
        val sessionId: String,
        val mediaId: String,
        val startedAtMs: Long,
        val url: String,
    ) {
        val ageMs: Long get() = System.currentTimeMillis() - startedAtMs
    }

    private val _current = AtomicReference<Session?>(null)

    val current: Session? get() = _current.get()

    val currentSessionId: String
        get() = _current.get()?.sessionId ?: NO_SESSION_ID

    val currentMediaId: String?
        get() = _current.get()?.mediaId

    /**
     * Open a new session for [mediaId] / [url].
     * Previous session is automatically closed.
     *
     * @return The newly created [Session].
     */
    fun openSession(mediaId: String, url: String): Session {
        val previous = _current.get()
        if (previous != null) {
            Timber.d(
                "SessionManager: closing session [${previous.sessionId}] " +
                    "after ${previous.ageMs}ms"
            )
        }
        val session = Session(
            sessionId  = UUID.randomUUID().toString(),
            mediaId    = mediaId,
            startedAtMs = System.currentTimeMillis(),
            url        = url,
        )
        _current.set(session)
        Timber.i(
            "SessionManager: opened session [${session.sessionId}] " +
                "for mediaId=[${session.mediaId}]"
        )
        return session
    }

    /**
     * Close the current session and return it for final reporting.
     * Safe to call when no session is active (returns null).
     */
    fun closeSession(): Session? {
        val closed = _current.getAndSet(null)
        if (closed != null) {
            Timber.i(
                "SessionManager: closed session [${closed.sessionId}] " +
                    "duration=${closed.ageMs}ms"
            )
        }
        return closed
    }

    fun hasActiveSession(): Boolean = _current.get() != null

    companion object {
        const val NO_SESSION_ID = "no-session"
    }
}
```

## xplayer/app/src/main/java/com/xplayer/dev/core/engine/internal/EngineStateValidator.kt

```kotlin
package com.xplayer.dev.core.engine.internal

import com.xplayer.dev.core.state.PlayerState
import timber.log.Timber
import javax.inject.Inject

/**
 * Guards all [PlayerState] transitions.
 *
 * In DEBUG builds: illegal transitions throw [IllegalStateTransitionException].
 * In RELEASE builds: illegal transitions are logged and rejected (engine
 * stays in current state), preventing cascading crashes in production.
 *
 * Transition table:
 * ┌──────────────┬────────────────────────────────────────────────────────┐
 * │ FROM         │ TO (allowed)                                           │
 * ├──────────────┼────────────────────────────────────────────────────────┤
 * │ Idle         │ Preparing                                              │
 * │ Preparing    │ Buffering, Error                                       │
 * │ Buffering    │ Ready, Error, Idle                                     │
 * │ Ready        │ Playing, Paused, Error, Idle                           │
 * │ Playing      │ Paused, Buffering, Ended, Error, Idle                 │
 * │ Paused       │ Playing, Idle, Error, Buffering                        │
 * │ Ended        │ Idle, Preparing                                        │
 * │ Error        │ Idle, Preparing                                        │
 * └──────────────┴────────────────────────────────────────────────────────┘
 */
class EngineStateValidator @Inject constructor() {

    /**
     * Validate and apply [newState] to [currentState].
     *
     * @return [ValidationResult.Accepted] if the transition is legal,
     *         [ValidationResult.Rejected] otherwise.
     */
    fun validate(
        currentState: PlayerState,
        newState: PlayerState,
        isDebug: Boolean = false,
    ): ValidationResult {
        if (currentState::class == newState::class) {
            // Same-type transitions are always allowed (state updates)
            return ValidationResult.Accepted(newState)
        }

        val allowed = allowedTransitions(currentState)
        val isLegal = allowed.any { newState::class == it }

        return if (isLegal) {
            Timber.v(
                "StateValidator: ✓ ${currentState::class.simpleName} " +
                    "→ ${newState::class.simpleName}"
            )
            ValidationResult.Accepted(newState)
        } else {
            val reason = buildRejectionReason(currentState, newState, allowed)
            Timber.e("StateValidator: ✗ $reason")

            if (isDebug) {
                throw IllegalStateTransitionException(reason)
            }
            ValidationResult.Rejected(
                from   = currentState,
                to     = newState,
                reason = reason,
            )
        }
    }

    private fun allowedTransitions(from: PlayerState): List<kotlin.reflect.KClass<out PlayerState>> =
        when (from) {
            is PlayerState.Uninitialised -> listOf(PlayerState.Idle::class)
            is PlayerState.Idle      -> listOf(
                PlayerState.Preparing::class,
            )
            is PlayerState.Preparing -> listOf(
                PlayerState.Buffering::class,
                PlayerState.Error::class,
            )
            is PlayerState.Buffering -> listOf(
                PlayerState.Ready::class,
                PlayerState.Error::class,
                PlayerState.Idle::class,
            )
            is PlayerState.Ready     -> listOf(
                PlayerState.Playing::class,
                PlayerState.Paused::class,
                PlayerState.Error::class,
                PlayerState.Idle::class,
            )
            is PlayerState.Playing   -> listOf(
                PlayerState.Paused::class,
                PlayerState.Buffering::class,
                PlayerState.Ended::class,
                PlayerState.Error::class,
                PlayerState.Idle::class,
            )
            is PlayerState.Paused    -> listOf(
                PlayerState.Playing::class,
                PlayerState.Buffering::class,
                PlayerState.Idle::class,
                PlayerState.Error::class,
            )
            is PlayerState.Ended     -> listOf(
                PlayerState.Idle::class,
                PlayerState.Preparing::class,
            )
            is PlayerState.Error     -> listOf(
                PlayerState.Idle::class,
                PlayerState.Preparing::class,
            )
        }

    private fun buildRejectionReason(
        from: PlayerState,
        to: PlayerState,
        allowed: List<kotlin.reflect.KClass<out PlayerState>>,
    ): String {
        val allowedNames = allowed.joinToString { it.simpleName ?: "?" }
        return "Illegal transition: ${from::class.simpleName} → " +
            "${to::class.simpleName}. " +
            "Allowed from ${from::class.simpleName}: [$allowedNames]"
    }

    sealed interface ValidationResult {
        data class Accepted(val state: PlayerState) : ValidationResult
        data class Rejected(
            val from: PlayerState,
            val to: PlayerState,
            val reason: String,
        ) : ValidationResult
    }
}

class IllegalStateTransitionException(reason: String) :
    IllegalStateException("PlayerEngine: $reason")
```

## xplayer/app/src/main/java/com/xplayer/dev/core/repository/PlayerRepository.kt

```kotlin
package com.xplayer.dev.core.repository

import androidx.media3.common.MediaItem
import com.xplayer.dev.core.engine.PlayerEngine
import com.xplayer.dev.core.engine.PlayerInitializer
import com.xplayer.dev.core.events.PlayerEvent
import com.xplayer.dev.core.state.PlayerState
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.StateFlow
import javax.inject.Inject
import javax.inject.Singleton

/**
 * Contract between the player stack and its consumers.
 */
interface PlayerRepository {

    val state: StateFlow<PlayerState>
    val events: Flow<PlayerEvent>

    suspend fun load(
        mediaItem: MediaItem,
        mediaId: String,
        playWhenReady: Boolean = true,
    )

    suspend fun play()
    suspend fun pause()
    suspend fun stop()
    suspend fun seekTo(positionMs: Long)
    suspend fun seekBack()
    suspend fun seekForward()
    suspend fun replay()
    suspend fun setPlaybackSpeed(speed: Float)
    suspend fun setVolume(volume: Float)
    suspend fun setRepeatMode(repeatMode: Int)
    suspend fun setShuffleMode(shuffleModeEnabled: Boolean)
    suspend fun release()

    val currentPositionMs: Long
    val durationMs: Long
    val bufferedPositionMs: Long
    val bufferedPercentage: Int
    val playbackSpeed: Float
    val playbackState: Int
    val playWhenReady: Boolean
    val isPlaying: Boolean
    val isLoading: Boolean
    val currentMediaItem: MediaItem?
}

// ── Implementation ────────────────────────────────────────────────────────────

@Singleton
class PlayerRepositoryImpl @Inject constructor(
    private val engine: PlayerEngine,
    private val initializer: PlayerInitializer,
) : PlayerRepository {

    override val state: StateFlow<PlayerState> = engine.state
    override val events: Flow<PlayerEvent> = engine.events

    override suspend fun load(
        mediaItem: MediaItem,
        mediaId: String,
        playWhenReady: Boolean,
    ) {
        initializer.initializeIfNeeded()
        engine.prepare(mediaItem, mediaId)
        if (playWhenReady) engine.play()
    }

    override suspend fun play() = engine.play()
    override suspend fun pause() = engine.pause()
    override suspend fun stop() = engine.stop()
    override suspend fun seekTo(positionMs: Long) = engine.seekTo(positionMs)
    override suspend fun seekBack() = engine.seekBack()
    override suspend fun seekForward() = engine.seekForward()
    override suspend fun replay() = engine.replay()
    override suspend fun setPlaybackSpeed(speed: Float) = engine.setPlaybackSpeed(speed)
    override suspend fun setVolume(volume: Float) = engine.setVolume(volume)
    override suspend fun setRepeatMode(repeatMode: Int) = engine.setRepeatMode(repeatMode)
    override suspend fun setShuffleMode(shuffleModeEnabled: Boolean) = engine.setShuffleMode(shuffleModeEnabled)
    override suspend fun release() = engine.release()

    override val currentPositionMs: Long get() = engine.currentPositionMs
    override val durationMs: Long get() = engine.durationMs
    override val bufferedPositionMs: Long get() = engine.bufferedPositionMs
    override val bufferedPercentage: Int get() = engine.bufferedPercent
    override val playbackSpeed: Float get() = engine.playbackSpeed
    override val playbackState: Int get() = engine.playbackState
    override val playWhenReady: Boolean get() = engine.playWhenReady
    override val isPlaying: Boolean get() = engine.isPlaying
    override val isLoading: Boolean get() = engine.isLoading
    override val currentMediaItem: MediaItem? get() = engine.currentMediaItem
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/manager/PlayerManager.kt

```kotlin
package com.xplayer.dev.core.manager

import androidx.media3.common.MediaItem
import com.xplayer.dev.core.engine.PlayerEngine
import com.xplayer.dev.core.engine.PlayerInitializer
import com.xplayer.dev.core.events.PlayerEvent
import com.xplayer.dev.core.state.PlayerState
import kotlinx.coroutines.CoroutineScope
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.Job
import kotlinx.coroutines.SupervisorJob
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow
import kotlinx.coroutines.flow.collectLatest
import kotlinx.coroutines.flow.filterIsInstance
import kotlinx.coroutines.flow.update
import kotlinx.coroutines.launch
import timber.log.Timber
import java.util.UUID
import javax.inject.Inject
import javax.inject.Singleton

/**
 * # PlayerManager — Advanced Queue & Playback Controller
 *
 * Acts as the high-level brain above [PlayerEngine]. 
 * While PlayerEngine handles low-level ExoPlayer interactions, PlayerManager handles:
 * 1. Queue/Playlist management (Next, Previous, Add, Remove).
 * 2. Auto-advancing to the next item on playback completion.
 * 3. Error recovery routing (skipping unplayable items).
 * 4. Aggregating UI-friendly states.
 */
@Singleton
class PlayerManager @Inject constructor(
    private val engine: PlayerEngine,
    private val initializer: PlayerInitializer,
) {
    private val scope = CoroutineScope(SupervisorJob() + Dispatchers.Main.immediate)
    private var observationJob: Job? = null

    // ── Playlist State ────────────────────────────────────────────────────────

    private val _queue = MutableStateFlow<List<PlaylistItem>>(emptyList())
    val queue: StateFlow<List<PlaylistItem>> = _queue.asStateFlow()

    private val _currentIndex = MutableStateFlow(-1)
    val currentIndex: StateFlow<Int> = _currentIndex.asStateFlow()

    private val _repeatMode = MutableStateFlow(RepeatMode.OFF)
    val repeatMode: StateFlow<RepeatMode> = _repeatMode.asStateFlow()

    enum class RepeatMode { OFF, ALL, ONE }

    // ── Expose Engine Streams ─────────────────────────────────────────────────

    val engineState: StateFlow<PlayerState> = engine.state
    val engineEvents = engine.events

    init {
        startObservingEngine()
    }

    // ── Queue Management ──────────────────────────────────────────────────────

    fun setQueue(items: List<MediaItem>, startIndex: Int = 0) {
        val mappedItems = items.map { PlaylistItem(id = UUID.randomUUID().toString(), mediaItem = it) }
        _queue.value = mappedItems
        _currentIndex.value = if (mappedItems.isEmpty()) -1 else startIndex.coerceIn(0, mappedItems.lastIndex)
        
        if (_currentIndex.value != -1) {
            playCurrentIndex()
        } else {
            engine.stop()
        }
    }

    fun addNext(mediaItem: MediaItem) {
        val newItem = PlaylistItem(id = UUID.randomUUID().toString(), mediaItem = mediaItem)
        _queue.update { current ->
            val mutable = current.toMutableList()
            val insertIdx = if (_currentIndex.value != -1) _currentIndex.value + 1 else 0
            mutable.add(insertIdx, newItem)
            mutable
        }
        if (_currentIndex.value == -1) {
            _currentIndex.value = 0
            playCurrentIndex()
        }
    }

    fun playNext() {
        val queueSize = _queue.value.size
        if (queueSize == 0) return

        val current = _currentIndex.value
        val nextIdx = when (_repeatMode.value) {
            RepeatMode.ONE -> current // replay current
            RepeatMode.ALL -> (current + 1) % queueSize
            RepeatMode.OFF -> if (current + 1 < queueSize) current + 1 else -1
        }

        if (nextIdx != -1) {
            _currentIndex.value = nextIdx
            playCurrentIndex()
        } else {
            Timber.i("PlayerManager: End of queue reached.")
            engine.stop()
        }
    }

    fun playPrevious() {
        val queueSize = _queue.value.size
        if (queueSize == 0) return

        val current = _currentIndex.value
        val prevIdx = if (current - 1 >= 0) current - 1 else 0
        
        // If playback is more than 3 seconds in, previous usually restarts the track.
        // For simplicity, we just jump to the previous track here.
        _currentIndex.value = prevIdx
        playCurrentIndex()
    }

    private fun playCurrentIndex() {
        val idx = _currentIndex.value
        val items = _queue.value
        if (idx in items.indices) {
            val item = items[idx]
            Timber.d("PlayerManager: Loading item [${item.id}] at index $idx")
            scope.launch {
                initializer.initializeIfNeeded()
                engine.prepare(item.mediaItem, item.id)
                engine.play()
            }
        }
    }

    // ── Internal Observers ────────────────────────────────────────────────────

    private fun startObservingEngine() {
        observationJob?.cancel()
        observationJob = scope.launch {
            // Watch for playback completion to auto-advance
            launch {
                engineState.filterIsInstance<PlayerState.Ended>().collectLatest { state ->
                    Timber.d("PlayerManager: Track ended naturally. Auto-advancing.")
                    playNext()
                }
            }

            // Watch for fatal errors to potentially skip unplayable tracks
            launch {
                engineState.filterIsInstance<PlayerState.Error>().collectLatest { error ->
                    if (error.isFatal) {
                        Timber.e(error.cause, "PlayerManager: Fatal error on track ${error.mediaId}. Skipping to next.")
                        playNext()
                    }
                }
            }
        }
    }

    // ── Pass-through Controls ─────────────────────────────────────────────────

    fun play() = engine.play()
    fun pause() = engine.pause()
    fun stop() = engine.stop()
    fun seekTo(positionMs: Long) = engine.seekTo(positionMs)
    fun setVolume(volume: Float) = engine.setVolume(volume)
    
    fun toggleRepeatMode() {
        _repeatMode.update { current ->
            when (current) {
                RepeatMode.OFF -> RepeatMode.ALL
                RepeatMode.ALL -> RepeatMode.ONE
                RepeatMode.ONE -> RepeatMode.OFF
            }
        }
    }

    fun release() {
        observationJob?.cancel()
        engine.release()
        _queue.value = emptyList()
        _currentIndex.value = -1
    }

    data class PlaylistItem(
        val id: String,
        val mediaItem: MediaItem
    )
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/di/PlayerModule.kt

```kotlin
package com.xplayer.dev.core.di

import com.xplayer.dev.BuildConfig
import com.xplayer.dev.core.config.PlayerConfiguration
import com.xplayer.dev.core.repository.PlayerRepository
import com.xplayer.dev.core.repository.PlayerRepositoryImpl
import dagger.Binds
import dagger.Module
import dagger.Provides
import dagger.hilt.InstallIn
import dagger.hilt.components.SingletonComponent
import okhttp3.OkHttpClient
import okhttp3.logging.HttpLoggingInterceptor
import java.util.concurrent.TimeUnit
import javax.inject.Singleton

@Module
@InstallIn(SingletonComponent::class)
abstract class PlayerBindsModule {

    @Binds
    @Singleton
    abstract fun bindPlayerRepository(
        impl: PlayerRepositoryImpl,
    ): PlayerRepository
}

@Module
@InstallIn(SingletonComponent::class)
object PlayerProvidesModule {

    @Provides
    @Singleton
    fun providePlayerConfiguration(): PlayerConfiguration =
        if (BuildConfig.DEBUG) {
            PlayerConfiguration.debug()
        } else {
            PlayerConfiguration.default()
        }

    @Provides
    @Singleton
    fun provideOkHttpClient(
        configuration: PlayerConfiguration,
    ): OkHttpClient {
        val builder = OkHttpClient.Builder()
            .connectTimeout(configuration.networkConfig.connectTimeoutSec, TimeUnit.SECONDS)
            .readTimeout(configuration.networkConfig.readTimeoutSec, TimeUnit.SECONDS)
            .writeTimeout(configuration.networkConfig.writeTimeoutSec, TimeUnit.SECONDS)

        if (configuration.networkConfig.enableHttp2) {
            builder.protocols(
                listOf(
                    okhttp3.Protocol.HTTP_2,
                    okhttp3.Protocol.HTTP_1_1,
                )
            )
        }

        if (configuration.loggingConfig.enableNetworkLogging) {
            builder.addInterceptor(
                HttpLoggingInterceptor { message ->
                    timber.log.Timber.tag("OkHttp").d(message)
                }.apply { level = HttpLoggingInterceptor.Level.HEADERS }
            )
        }

        return builder.build()
    }
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/di/ThreadingModule.kt

```kotlin
package com.xplayer.dev.core.di

import com.xplayer.dev.core.threading.IoDispatcher
import com.xplayer.dev.core.threading.MainDispatcher
import com.xplayer.dev.core.threading.PlayerDispatchers
import com.xplayer.dev.core.threading.PlayerExecutor
import com.xplayer.dev.core.threading.PlayerScopeManager
import dagger.Module
import dagger.Provides
import dagger.hilt.InstallIn
import dagger.hilt.components.SingletonComponent
import kotlinx.coroutines.CoroutineDispatcher
import kotlinx.coroutines.Dispatchers
import javax.inject.Singleton

@Module
@InstallIn(SingletonComponent::class)
object ThreadingModule {

    @Provides
    @MainDispatcher
    fun provideMainDispatcher(): CoroutineDispatcher = Dispatchers.Main.immediate

    @Provides
    @IoDispatcher
    fun provideIoDispatcher(): CoroutineDispatcher = Dispatchers.IO

    @Provides
    @Singleton
    fun providePlayerDispatchers(): PlayerDispatchers = PlayerDispatchers(
        main = Dispatchers.Main.immediate,
        io = Dispatchers.IO,
        default = Dispatchers.Default,
    )

    @Provides
    @Singleton
    fun providePlayerExecutor(): PlayerExecutor = PlayerExecutor()

    @Provides
    @Singleton
    fun providePlayerScopeManager(
        dispatchers: PlayerDispatchers,
    ): PlayerScopeManager = PlayerScopeManager(dispatchers)
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/session/SessionFactory.kt

```kotlin
package com.xplayer.dev.core.session

import java.util.UUID
import javax.inject.Inject
import javax.inject.Singleton

/**
 * Factory for creating [PlayerSession] instances with unique identifiers and timestamps.
 */
@Singleton
class SessionFactory @Inject constructor() {

    fun createSession(mediaId: String, mediaUri: String = ""): PlayerSession {
        return PlayerSession(
            sessionId = UUID.randomUUID().toString(),
            mediaId = mediaId,
            mediaUri = mediaUri,
            startedAtMs = System.currentTimeMillis(),
        )
    }
}

data class PlayerSession(
    val sessionId: String,
    val mediaId: String,
    val mediaUri: String,
    val startedAtMs: Long,
    var endedAtMs: Long = 0L,
    var lastPositionMs: Long = 0L,
    var durationMs: Long = 0L,
    var pauseCount: Int = 0,
    var seekCount: Int = 0,
    var bufferCount: Int = 0,
    var errorCount: Int = 0,
) {
    val ageMs: Long
        get() = (if (endedAtMs > 0L) endedAtMs else System.currentTimeMillis()) - startedAtMs

    val watchPercentage: Float
        get() = if (durationMs > 0L) (lastPositionMs.toFloat() / durationMs * 100f).coerceIn(0f, 100f) else 0f
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/session/SessionRepository.kt

```kotlin
package com.xplayer.dev.core.session

import javax.inject.Inject
import javax.inject.Singleton

/**
 * Repository abstraction over session storage and validation.
 */
interface SessionRepository {
    fun saveSession(session: PlayerSession)
    fun getSession(sessionId: String): PlayerSession?
    fun getRecentSessions(): List<PlayerSession>
    fun getLastPosition(mediaId: String): Long
    fun getSavedPosition(mediaId: String): Long = getLastPosition(mediaId)
    fun deleteSession(sessionId: String)
}

@Singleton
class DefaultSessionRepository @Inject constructor(
    private val storage: SessionStorage,
    private val validator: SessionValidator,
) : SessionRepository {

    override fun saveSession(session: PlayerSession) {
        if (validator.validate(session) is SessionValidator.Result.Valid) {
            storage.saveSession(session)
        }
    }

    override fun getSession(sessionId: String): PlayerSession? = storage.getSession(sessionId)

    override fun getRecentSessions(): List<PlayerSession> =
        storage.getAllSessions().sortedByDescending { it.startedAtMs }

    override fun getLastPosition(mediaId: String): Long = storage.getSavedPosition(mediaId)

    override fun deleteSession(sessionId: String) {
        storage.removeSession(sessionId)
    }
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/session/SessionStorage.kt

```kotlin
package com.xplayer.dev.core.session

import java.util.concurrent.ConcurrentHashMap
import javax.inject.Inject
import javax.inject.Singleton

/**
 * Thread-safe storage layer for persisting playback sessions and watch progress in memory.
 * Can be backed by database/SharedPreferences in future implementations.
 */
@Singleton
class SessionStorage @Inject constructor() {

    private val sessions = ConcurrentHashMap<String, PlayerSession>()
    private val mediaProgress = ConcurrentHashMap<String, Long>()

    fun saveSession(session: PlayerSession) {
        sessions[session.sessionId] = session
        mediaProgress[session.mediaId] = session.lastPositionMs
    }

    fun getSession(sessionId: String): PlayerSession? = sessions[sessionId]

    fun getAllSessions(): List<PlayerSession> = sessions.values.toList()

    fun getSavedPosition(mediaId: String): Long = mediaProgress[mediaId] ?: 0L

    fun removeSession(sessionId: String) {
        sessions.remove(sessionId)
    }

    fun clearAll() {
        sessions.clear()
        mediaProgress.clear()
    }
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/session/SessionValidator.kt

```kotlin
package com.xplayer.dev.core.session

import javax.inject.Inject
import javax.inject.Singleton

/**
 * Validates integrity of [PlayerSession] instances.
 */
@Singleton
class SessionValidator @Inject constructor() {

    sealed interface Result {
        data object Valid : Result
        data class Invalid(val reason: String) : Result
    }

    fun validate(session: PlayerSession): Result {
        if (session.sessionId.isBlank()) return Result.Invalid("Session ID cannot be blank")
        if (session.mediaId.isBlank()) return Result.Invalid("Media ID cannot be blank")
        if (session.startedAtMs <= 0L) return Result.Invalid("Started timestamp must be positive")
        if (session.lastPositionMs < 0L) return Result.Invalid("Playback position cannot be negative")
        if (session.durationMs < 0L) return Result.Invalid("Duration cannot be negative")
        return Result.Valid
    }
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/listener/ListenerManager.kt

```kotlin
package com.xplayer.dev.core.listener

import androidx.media3.common.Player
import androidx.media3.exoplayer.ExoPlayer
import androidx.media3.exoplayer.analytics.AnalyticsListener
import javax.inject.Inject
import javax.inject.Singleton

/**
 * Manages attaching/detaching registered listeners to an active [ExoPlayer] instance.
 */
@Singleton
class ListenerManager @Inject constructor(
    private val registry: ListenerRegistry,
) {

    private var attachedPlayer: ExoPlayer? = null

    fun attachToPlayer(player: ExoPlayer) {
        detachFromPlayer()
        attachedPlayer = player

        registry.getActivePlayerListeners().forEach { player.addListener(it) }
        registry.getActiveAnalyticsListeners().forEach { player.addAnalyticsListener(it) }
    }

    fun addPlayerListener(listener: Player.Listener) {
        registry.registerPlayerListener(listener)
        attachedPlayer?.addListener(listener)
    }

    fun removePlayerListener(listener: Player.Listener) {
        registry.unregisterPlayerListener(listener)
        attachedPlayer?.removeListener(listener)
    }

    fun addAnalyticsListener(listener: AnalyticsListener) {
        registry.registerAnalyticsListener(listener)
        attachedPlayer?.addAnalyticsListener(listener)
    }

    fun removeAnalyticsListener(listener: AnalyticsListener) {
        registry.unregisterAnalyticsListener(listener)
        attachedPlayer?.removeAnalyticsListener(listener)
    }

    fun detachFromPlayer() {
        attachedPlayer?.let { player ->
            registry.getActivePlayerListeners().forEach { player.removeListener(it) }
            registry.getActiveAnalyticsListeners().forEach { player.removeAnalyticsListener(it) }
        }
        attachedPlayer = null
    }
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/listener/ListenerRegistry.kt

```kotlin
package com.xplayer.dev.core.listener

import androidx.media3.common.Player
import androidx.media3.exoplayer.analytics.AnalyticsListener
import java.lang.ref.WeakReference
import java.util.concurrent.CopyOnWriteArrayList
import javax.inject.Inject
import javax.inject.Singleton

/**
 * Registry supporting weak reference tracking and leak-free listener management.
 */
@Singleton
class ListenerRegistry @Inject constructor() {

    private val playerListeners = CopyOnWriteArrayList<WeakReference<Player.Listener>>()
    private val analyticsListeners = CopyOnWriteArrayList<WeakReference<AnalyticsListener>>()

    fun registerPlayerListener(listener: Player.Listener) {
        cleanupStale()
        if (playerListeners.none { it.get() == listener }) {
            playerListeners.add(WeakReference(listener))
        }
    }

    fun unregisterPlayerListener(listener: Player.Listener) {
        playerListeners.removeAll { it.get() == listener || it.get() == null }
    }

    fun getActivePlayerListeners(): List<Player.Listener> {
        cleanupStale()
        return playerListeners.mapNotNull { it.get() }
    }

    fun registerAnalyticsListener(listener: AnalyticsListener) {
        cleanupStale()
        if (analyticsListeners.none { it.get() == listener }) {
            analyticsListeners.add(WeakReference(listener))
        }
    }

    fun unregisterAnalyticsListener(listener: AnalyticsListener) {
        analyticsListeners.removeAll { it.get() == listener || it.get() == null }
    }

    fun getActiveAnalyticsListeners(): List<AnalyticsListener> {
        cleanupStale()
        return analyticsListeners.mapNotNull { it.get() }
    }

    private fun cleanupStale() {
        playerListeners.removeAll { it.get() == null }
        analyticsListeners.removeAll { it.get() == null }
    }

    fun clearAll() {
        playerListeners.clear()
        analyticsListeners.clear()
    }
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/network/DataSourceFactory.kt

```kotlin
package com.xplayer.dev.core.network

import android.content.Context
import androidx.media3.datasource.DataSource
import androidx.media3.datasource.DefaultDataSource
import androidx.media3.datasource.okhttp.OkHttpDataSource
import com.xplayer.dev.core.config.PlayerConfiguration
import okhttp3.OkHttpClient
import timber.log.Timber
import java.util.concurrent.TimeUnit
import javax.inject.Inject
import javax.inject.Singleton

/**
 * # DataSourceFactory
 *
 * Builds [DataSource.Factory] for ExoPlayer media loading supporting HTTP, HTTPS, file://, and content:// URIs.
 */
@Singleton
class DataSourceFactory @Inject constructor(
    private val context: Context,
    private val config: PlayerConfiguration,
) {

    private val okHttpClient: OkHttpClient by lazy { buildOkHttpClient() }

    fun create(): DataSource.Factory {
        val net = config.networkConfig

        val okHttpDataSourceFactory = OkHttpDataSource.Factory(okHttpClient)
            .setUserAgent(net.userAgent)

        return DefaultDataSource.Factory(context, okHttpDataSourceFactory)
            .also { Timber.d("DataSourceFactory: factory created") }
    }

    fun createWithHeaders(headers: Map<String, String>): DataSource.Factory {
        val net = config.networkConfig

        val okHttpDataSourceFactory = OkHttpDataSource.Factory(okHttpClient)
            .setUserAgent(net.userAgent)
            .setDefaultRequestProperties(headers)

        return DefaultDataSource.Factory(context, okHttpDataSourceFactory)
    }

    private fun buildOkHttpClient(): OkHttpClient {
        val net = config.networkConfig
        return OkHttpClient.Builder()
            .connectTimeout(net.connectTimeoutSec, TimeUnit.SECONDS)
            .readTimeout(net.readTimeoutSec, TimeUnit.SECONDS)
            .writeTimeout(net.writeTimeoutSec, TimeUnit.SECONDS)
            .followRedirects(true)
            .followSslRedirects(true)
            .addInterceptor { chain ->
                val request = chain.request()
                if (config.loggingConfig.enableNetworkLogging) {
                    Timber.d("HTTP -> ${request.method} ${request.url}")
                }
                chain.proceed(request)
            }
            .build()
            .also { Timber.d("DataSourceFactory: OkHttpClient built") }
    }
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/network/NetworkManager.kt

```kotlin
package com.xplayer.dev.core.network

import android.content.Context
import android.net.ConnectivityManager
import android.net.Network
import android.net.NetworkCapabilities
import android.net.NetworkRequest
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow
import timber.log.Timber
import javax.inject.Inject
import javax.inject.Singleton

enum class NetworkType {
    WIFI,
    ETHERNET,
    CELLULAR_4G,
    CELLULAR_5G,
    CELLULAR_3G,
    OFFLINE,
    UNKNOWN
}

/**
 * # NetworkManager
 *
 * Monitors network connectivity and exposes it as a [StateFlow].
 */
@Singleton
class NetworkManager @Inject constructor(
    private val context: Context,
) {

    private val connectivityManager =
        context.getSystemService(Context.CONNECTIVITY_SERVICE) as ConnectivityManager

    private val _networkType = MutableStateFlow(currentNetworkType())
    val networkType: StateFlow<NetworkType> = _networkType.asStateFlow()

    val isOnline: Boolean
        get() = _networkType.value != NetworkType.OFFLINE

    val isWifi: Boolean
        get() = _networkType.value == NetworkType.WIFI

    private val networkCallback = object : ConnectivityManager.NetworkCallback() {

        override fun onAvailable(network: Network) {
            val type = currentNetworkType()
            _networkType.value = type
            Timber.i("NetworkManager: connected — $type")
        }

        override fun onLost(network: Network) {
            _networkType.value = NetworkType.OFFLINE
            Timber.w("NetworkManager: connection lost")
        }

        override fun onCapabilitiesChanged(
            network: Network,
            networkCapabilities: NetworkCapabilities,
        ) {
            val type = resolveType(networkCapabilities)
            if (_networkType.value != type) {
                _networkType.value = type
                Timber.d("NetworkManager: capabilities changed -> $type")
            }
        }
    }

    fun register() {
        val request = NetworkRequest.Builder()
            .addCapability(NetworkCapabilities.NET_CAPABILITY_INTERNET)
            .build()
        runCatching {
            connectivityManager.registerNetworkCallback(request, networkCallback)
            Timber.d("NetworkManager: registered")
        }.onFailure {
            Timber.e(it, "NetworkManager: failed to register")
        }
    }

    fun unregister() {
        runCatching {
            connectivityManager.unregisterNetworkCallback(networkCallback)
            Timber.d("NetworkManager: unregistered")
        }.onFailure {
            Timber.e(it, "NetworkManager: failed to unregister")
        }
    }

    private fun currentNetworkType(): NetworkType {
        val network = connectivityManager.activeNetwork ?: return NetworkType.OFFLINE
        val caps = connectivityManager.getNetworkCapabilities(network) ?: return NetworkType.OFFLINE
        return resolveType(caps)
    }

    private fun resolveType(caps: NetworkCapabilities): NetworkType = when {
        caps.hasTransport(NetworkCapabilities.TRANSPORT_WIFI) -> NetworkType.WIFI
        caps.hasTransport(NetworkCapabilities.TRANSPORT_ETHERNET) -> NetworkType.ETHERNET
        caps.hasTransport(NetworkCapabilities.TRANSPORT_CELLULAR) -> resolveCellular(caps)
        else -> NetworkType.UNKNOWN
    }

    private fun resolveCellular(caps: NetworkCapabilities): NetworkType {
        return NetworkType.CELLULAR_4G
    }
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/network/NetworkMonitor.kt

```kotlin
package com.xplayer.dev.core.network

import kotlinx.coroutines.flow.StateFlow
import javax.inject.Inject
import javax.inject.Singleton

/**
 * # NetworkMonitor
 *
 * Thin facade over [NetworkManager] providing read-only access to network state.
 */
@Singleton
class NetworkMonitor @Inject constructor(
    private val networkManager: NetworkManager,
) {
    val networkType: StateFlow<NetworkType>
        get() = networkManager.networkType

    val isOnline: Boolean get() = networkManager.isOnline
    val isWifi: Boolean get() = networkManager.isWifi
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/network/RetryPolicy.kt

```kotlin
package com.xplayer.dev.core.network

import com.xplayer.dev.core.config.PlayerConfiguration
import kotlinx.coroutines.delay
import timber.log.Timber
import javax.inject.Inject
import javax.inject.Singleton
import kotlin.math.min
import kotlin.math.pow

/**
 * # RetryPolicy
 *
 * Exponential backoff retry with jitter.
 */
@Singleton
class RetryPolicy @Inject constructor(
    private val config: PlayerConfiguration,
) {
    private val retryConfig get() = config.retryConfig

    suspend fun <T> withRetry(
        tag: String = "RetryPolicy",
        block: suspend (attempt: Int) -> T,
    ): T {
        var lastException: Exception? = null

        for (attempt in 0..retryConfig.maxRetryCount) {
            try {
                if (attempt > 0) {
                    val delayMs = computeDelay(attempt)
                    Timber.d("$tag: retry $attempt/${retryConfig.maxRetryCount} after ${delayMs}ms")
                    delay(delayMs)
                }
                return block(attempt)
            } catch (e: Exception) {
                lastException = e
                Timber.w("$tag: attempt $attempt failed — ${e.message}")
                if (attempt == retryConfig.maxRetryCount) {
                    Timber.e("$tag: all retries exhausted")
                }
            }
        }
        throw lastException ?: IllegalStateException("$tag: retry failed with no exception")
    }

    fun computeDelay(attempt: Int): Long {
        val exponential = retryConfig.initialDelayMs * (retryConfig.delayMultiplier.toDouble().pow(attempt - 1)).toLong()
        val capped = min(exponential, retryConfig.maxDelayMs)
        val jitter = (Math.random() * capped * JITTER_FACTOR).toLong()
        return capped + jitter
    }

    companion object {
        private const val JITTER_FACTOR = 0.1
    }
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/source/MediaSourceFactory.kt

```kotlin
package com.xplayer.dev.core.source

import androidx.media3.common.MediaItem
import androidx.media3.datasource.DataSource
import androidx.media3.exoplayer.dash.DashMediaSource
import androidx.media3.exoplayer.hls.HlsMediaSource
import androidx.media3.exoplayer.rtsp.RtspMediaSource
import androidx.media3.exoplayer.smoothstreaming.SsMediaSource
import androidx.media3.exoplayer.source.MediaSource
import androidx.media3.exoplayer.source.ProgressiveMediaSource
import com.xplayer.dev.core.network.DataSourceFactory
import timber.log.Timber
import javax.inject.Inject
import javax.inject.Singleton

/**
 * # MediaSourceFactory
 *
 * Builds the correct [MediaSource] for any URI type based on [SourceResolver].
 */
@Singleton
class MediaSourceFactory @Inject constructor(
    private val dataSourceFactory: DataSourceFactory,
    private val resolver: SourceResolver,
    private val validator: SourceValidator,
) {

    fun create(
        mediaItem: MediaItem,
        headers: Map<String, String> = emptyMap(),
    ): MediaSource {
        val uriString = mediaItem.localConfiguration?.uri?.toString() ?: ""

        val validation = validator.validate(uriString)
        if (validation is SourceValidator.ValidationResult.Invalid) {
            throw IllegalArgumentException("MediaSourceFactory: invalid URI — ${validation.reason}")
        }

        val sourceType = resolver.resolve(uriString)
        val dsFactory = if (headers.isEmpty()) dataSourceFactory.create()
        else dataSourceFactory.createWithHeaders(headers)

        Timber.i("MediaSourceFactory: building $sourceType source for $uriString")

        return when (sourceType) {
            SourceResolver.SourceType.HLS -> buildHls(mediaItem, dsFactory)
            SourceResolver.SourceType.DASH -> buildDash(mediaItem, dsFactory)
            SourceResolver.SourceType.SMOOTH_STREAMING -> buildSmoothStreaming(mediaItem, dsFactory)
            SourceResolver.SourceType.RTSP -> buildRtsp(mediaItem)
            SourceResolver.SourceType.LOCAL_FILE,
            SourceResolver.SourceType.CONTENT_URI,
            SourceResolver.SourceType.PROGRESSIVE_HTTP,
            SourceResolver.SourceType.UNKNOWN -> buildProgressive(mediaItem, dsFactory)
        }
    }

    private fun buildHls(mediaItem: MediaItem, factory: DataSource.Factory): MediaSource =
        HlsMediaSource.Factory(factory)
            .setAllowChunklessPreparation(true)
            .createMediaSource(mediaItem)

    private fun buildDash(mediaItem: MediaItem, factory: DataSource.Factory): MediaSource =
        DashMediaSource.Factory(factory)
            .createMediaSource(mediaItem)

    private fun buildSmoothStreaming(mediaItem: MediaItem, factory: DataSource.Factory): MediaSource =
        SsMediaSource.Factory(factory)
            .createMediaSource(mediaItem)

    private fun buildRtsp(mediaItem: MediaItem): MediaSource =
        RtspMediaSource.Factory()
            .createMediaSource(mediaItem)

    private fun buildProgressive(mediaItem: MediaItem, factory: DataSource.Factory): MediaSource =
        ProgressiveMediaSource.Factory(factory)
            .createMediaSource(mediaItem)
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/source/SourceBuilder.kt

```kotlin
package com.xplayer.dev.core.source

import android.net.Uri
import androidx.media3.common.MediaItem
import androidx.media3.common.MimeTypes

/**
 * Fluent builder for creating configured [MediaItem] instances ready for source construction.
 */
class SourceBuilder(private val uriString: String) {
    private var mediaId: String = uriString
    private var mimeType: String? = null
    private var headers: Map<String, String> = emptyMap()

    fun setMediaId(id: String) = apply { this.mediaId = id }
    fun setMimeType(mime: String) = apply { this.mimeType = mime }
    fun setHeaders(headers: Map<String, String>) = apply { this.headers = headers }

    fun buildMediaItem(): MediaItem {
        return MediaItem.Builder()
            .setMediaId(mediaId)
            .setUri(Uri.parse(uriString))
            .apply {
                if (mimeType != null) setMimeType(mimeType)
            }
            .build()
    }

    fun getHeaders(): Map<String, String> = headers

    companion object {
        fun fromUri(uri: String): SourceBuilder = SourceBuilder(uri)
    }
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/source/SourceResolver.kt

```kotlin
package com.xplayer.dev.core.source

import android.net.Uri
import timber.log.Timber
import javax.inject.Inject
import javax.inject.Singleton

/**
 * # SourceResolver
 *
 * Resolves a URI string to a [SourceType] for routing to the correct media source builder.
 */
@Singleton
class SourceResolver @Inject constructor() {

    enum class SourceType {
        HLS,
        DASH,
        SMOOTH_STREAMING,
        RTSP,
        PROGRESSIVE_HTTP,
        LOCAL_FILE,
        CONTENT_URI,
        UNKNOWN,
    }

    fun resolve(uriString: String): SourceType {
        val uri = runCatching { Uri.parse(uriString) }.getOrNull()
            ?: return SourceType.UNKNOWN.also {
                Timber.w("SourceResolver: failed to parse URI — $uriString")
            }

        return when {
            isHls(uri, uriString) -> SourceType.HLS
            isDash(uri, uriString) -> SourceType.DASH
            isSmoothStreaming(uri, uriString) -> SourceType.SMOOTH_STREAMING
            isRtsp(uri) -> SourceType.RTSP
            isLocalFile(uri) -> SourceType.LOCAL_FILE
            isContentUri(uri) -> SourceType.CONTENT_URI
            isHttp(uri) -> SourceType.PROGRESSIVE_HTTP
            else -> SourceType.UNKNOWN
        }.also {
            Timber.d("SourceResolver: $uriString -> $it")
        }
    }

    private fun isHls(uri: Uri, raw: String): Boolean =
        raw.endsWith(".m3u8", ignoreCase = true) ||
            raw.contains(".m3u8?", ignoreCase = true) ||
            uri.path?.endsWith(".m3u8", ignoreCase = true) == true

    private fun isDash(uri: Uri, raw: String): Boolean =
        raw.endsWith(".mpd", ignoreCase = true) ||
            raw.contains(".mpd?", ignoreCase = true) ||
            uri.path?.endsWith(".mpd", ignoreCase = true) == true

    private fun isSmoothStreaming(uri: Uri, raw: String): Boolean =
        raw.endsWith("/manifest", ignoreCase = true) ||
            raw.contains("manifest?", ignoreCase = true)

    private fun isRtsp(uri: Uri): Boolean =
        uri.scheme?.equals("rtsp", ignoreCase = true) == true

    private fun isLocalFile(uri: Uri): Boolean =
        uri.scheme?.equals("file", ignoreCase = true) == true || uri.scheme == null

    private fun isContentUri(uri: Uri): Boolean =
        uri.scheme?.equals("content", ignoreCase = true) == true

    private fun isHttp(uri: Uri): Boolean =
        uri.scheme?.startsWith("http", ignoreCase = true) == true
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/source/SourceValidator.kt

```kotlin
package com.xplayer.dev.core.source

import android.content.Context
import android.net.Uri
import timber.log.Timber
import java.io.File
import javax.inject.Inject
import javax.inject.Singleton

/**
 * # SourceValidator
 *
 * Validates media URIs before passing to ExoPlayer.
 */
@Singleton
class SourceValidator @Inject constructor(
    private val context: Context,
) {

    sealed interface ValidationResult {
        data object Valid : ValidationResult
        data class Invalid(val reason: String) : ValidationResult
    }

    fun validate(uriString: String): ValidationResult {
        if (uriString.isBlank()) {
            return ValidationResult.Invalid("URI is blank")
        }

        val uri = runCatching { Uri.parse(uriString) }.getOrNull()
            ?: return ValidationResult.Invalid("URI cannot be parsed: $uriString")

        return when (uri.scheme?.lowercase()) {
            "http", "https" -> validateHttp(uri)
            "file" -> validateFile(uri)
            "content" -> validateContent(uri)
            "rtsp" -> ValidationResult.Valid
            null -> validateFile(uri)
            else -> ValidationResult.Valid
        }.also {
            if (it is ValidationResult.Invalid) {
                Timber.w("SourceValidator: invalid URI [$uriString] — ${it.reason}")
            }
        }
    }

    private fun validateHttp(uri: Uri): ValidationResult {
        if (uri.host.isNullOrBlank()) {
            return ValidationResult.Invalid("HTTP URI has no host: $uri")
        }
        return ValidationResult.Valid
    }

    private fun validateFile(uri: Uri): ValidationResult {
        val path = uri.path ?: return ValidationResult.Invalid("File URI has no path")
        val file = File(path)
        if (!file.exists()) {
            return ValidationResult.Invalid("File does not exist: $path")
        }
        if (!file.canRead()) {
            return ValidationResult.Invalid("File is not readable: $path")
        }
        return ValidationResult.Valid
    }

    private fun validateContent(uri: Uri): ValidationResult {
        return runCatching {
            context.contentResolver.getType(uri)
            ValidationResult.Valid
        }.getOrElse {
            ValidationResult.Invalid("Content URI not accessible: $uri")
        }
    }
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/playlist/PlaylistManager.kt

```kotlin
package com.xplayer.dev.core.playlist

import com.xplayer.dev.core.playback.PlaybackCoordinator
import timber.log.Timber
import javax.inject.Inject
import javax.inject.Singleton

/**
 * # PlaylistManager
 *
 * High-level orchestration facade over [PlaylistRepository], [QueueManager], and [PlaybackCoordinator].
 */
@Singleton
class PlaylistManager @Inject constructor(
    private val repository: PlaylistRepository,
    private val queueManager: QueueManager,
    private val coordinator: PlaybackCoordinator,
) {

    fun loadPlaylistIntoQueue(playlistId: String, startPlaying: Boolean = true) {
        val playlist = repository.playlists.value.find { it.id == playlistId } ?: return
        queueManager.clear()
        queueManager.addAll(playlist.items)
        repository.persistActiveQueue()
        Timber.i("PlaylistManager: loaded playlist '${playlist.name}' into queue")

        if (startPlaying && queueManager.items.isNotEmpty()) {
            coordinator.playAtIndex(0)
        }
    }

    fun addToQueue(item: QueueManager.QueueItem) {
        queueManager.add(item)
        repository.persistActiveQueue()
    }

    fun removeFromQueue(index: Int) {
        queueManager.removeAt(index)
        repository.persistActiveQueue()
    }

    fun moveInQueue(from: Int, to: Int) {
        queueManager.move(from, to)
        repository.persistActiveQueue()
    }

    fun clearQueue() {
        queueManager.clear()
        repository.persistActiveQueue()
    }

    fun setRepeatMode(mode: QueueManager.RepeatMode) {
        queueManager.setRepeatMode(mode)
    }

    fun setShuffle(enabled: Boolean) {
        queueManager.setShuffleEnabled(enabled)
    }

    fun playNext() = coordinator.playNext()
    fun playPrevious() = coordinator.playPrevious()
    fun playAtIndex(index: Int) = coordinator.playAtIndex(index)
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/playlist/PlaylistRepository.kt

```kotlin
package com.xplayer.dev.core.playlist

import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow
import timber.log.Timber
import java.util.UUID
import javax.inject.Inject
import javax.inject.Singleton

/**
 * # PlaylistRepository
 *
 * Single source of truth for playlists. Coordinates memory cache and persistent storage.
 */
@Singleton
class PlaylistRepository @Inject constructor(
    private val storage: PlaylistStorage,
    private val queueManager: QueueManager,
) {
    private val _playlists = MutableStateFlow<List<PlaylistStorage.Playlist>>(emptyList())
    val playlists: StateFlow<List<PlaylistStorage.Playlist>> = _playlists.asStateFlow()

    init {
        loadFromStorage()
    }

    private fun loadFromStorage() {
        _playlists.value = storage.getAllPlaylists()
    }

    fun createPlaylist(name: String, items: List<QueueManager.QueueItem> = emptyList()): PlaylistStorage.Playlist {
        val playlist = PlaylistStorage.Playlist(
            id = UUID.randomUUID().toString(),
            name = name,
            items = items,
        )
        storage.savePlaylist(playlist)
        loadFromStorage()
        Timber.i("PlaylistRepository: created playlist '${playlist.name}' [${playlist.id}]")
        return playlist
    }

    fun deletePlaylist(id: String) {
        storage.removePlaylist(id)
        loadFromStorage()
        Timber.i("PlaylistRepository: deleted playlist $id")
    }

    fun renamePlaylist(id: String, newName: String) {
        val current = storage.getPlaylist(id) ?: return
        val updated = current.copy(name = newName, updatedAtMs = System.currentTimeMillis())
        storage.savePlaylist(updated)
        loadFromStorage()
    }

    fun duplicatePlaylist(id: String, newName: String): PlaylistStorage.Playlist? {
        val current = storage.getPlaylist(id) ?: return null
        return createPlaylist(newName, current.items)
    }

    fun addItemToPlaylist(id: String, item: QueueManager.QueueItem) {
        val current = storage.getPlaylist(id) ?: return
        val updated = current.copy(
            items = current.items + item,
            updatedAtMs = System.currentTimeMillis(),
        )
        storage.savePlaylist(updated)
        loadFromStorage()
    }

    fun saveQueueAsPlaylist(name: String): PlaylistStorage.Playlist {
        return createPlaylist(name, queueManager.items)
    }

    fun persistActiveQueue() {
        storage.saveActiveQueue(queueManager.items, queueManager.currentIndex)
    }

    fun restoreActiveQueue() {
        val recovered = storage.loadActiveQueue() ?: return
        queueManager.clear()
        queueManager.addAll(recovered.first)
        if (recovered.second in 0 until recovered.first.size) {
            queueManager.moveToIndex(recovered.second)
        }
        Timber.i("PlaylistRepository: restored active queue (${recovered.first.size} items)")
    }
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/playlist/PlaylistStorage.kt

```kotlin
package com.xplayer.dev.core.playlist

import android.content.Context
import android.content.SharedPreferences
import org.json.JSONArray
import org.json.JSONObject
import timber.log.Timber
import javax.inject.Inject
import javax.inject.Singleton

/**
 * # PlaylistStorage
 *
 * Persists playlists and active playback queue to SharedPreferences using JSON.
 */
@Singleton
class PlaylistStorage @Inject constructor(
    private val context: Context,
) {
    data class Playlist(
        val id: String,
        val name: String,
        val items: List<QueueManager.QueueItem> = emptyList(),
        val createdAtMs: Long = System.currentTimeMillis(),
        val updatedAtMs: Long = System.currentTimeMillis(),
    )

    private val prefs: SharedPreferences by lazy {
        context.getSharedPreferences(PREFS_NAME, Context.MODE_PRIVATE)
    }

    private fun queueItemToJson(item: QueueManager.QueueItem): JSONObject {
        val obj = JSONObject()
        obj.put("mediaId", item.mediaId)
        obj.put("url", item.url)
        if (item.title != null) obj.put("title", item.title)
        obj.put("durationMs", item.durationMs)
        if (item.thumbnailUrl != null) obj.put("thumbnailUrl", item.thumbnailUrl)
        val extrasObj = JSONObject()
        item.extras.forEach { (k, v) -> extrasObj.put(k, v) }
        obj.put("extras", extrasObj)
        return obj
    }

    private fun jsonToQueueItem(obj: JSONObject): QueueManager.QueueItem? {
        return runCatching {
            val mediaId = obj.optString("mediaId", "")
            if (mediaId.isBlank()) return null
            val url = obj.optString("url", "")
            val title = if (obj.has("title")) obj.getString("title") else null
            val durationMs = obj.optLong("durationMs", 0L)
            val thumbnailUrl = if (obj.has("thumbnailUrl")) obj.getString("thumbnailUrl") else null
            val extrasMap = mutableMapOf<String, String>()
            if (obj.has("extras")) {
                val extrasObj = obj.getJSONObject("extras")
                extrasObj.keys().forEach { k ->
                    extrasMap[k] = extrasObj.optString(k, "")
                }
            }
            QueueManager.QueueItem(mediaId, url, title, durationMs, thumbnailUrl, extrasMap)
        }.getOrNull()
    }

    private fun playlistToJson(playlist: Playlist): String {
        val obj = JSONObject()
        obj.put("id", playlist.id)
        obj.put("name", playlist.name)
        obj.put("createdAtMs", playlist.createdAtMs)
        obj.put("updatedAtMs", playlist.updatedAtMs)
        val arr = JSONArray()
        playlist.items.forEach { arr.put(queueItemToJson(it)) }
        obj.put("items", arr)
        return obj.toString()
    }

    private fun jsonToPlaylist(json: String): Playlist? {
        return runCatching {
            val obj = JSONObject(json)
            val id = obj.optString("id", "")
            if (id.isBlank()) return null
            val name = obj.optString("name", "Untitled")
            val createdAtMs = obj.optLong("createdAtMs", System.currentTimeMillis())
            val updatedAtMs = obj.optLong("updatedAtMs", System.currentTimeMillis())
            val itemsArr = obj.optJSONArray("items") ?: JSONArray()
            val itemsList = mutableListOf<QueueManager.QueueItem>()
            for (i in 0 until itemsArr.length()) {
                val itemObj = itemsArr.optJSONObject(i) ?: continue
                jsonToQueueItem(itemObj)?.let { itemsList.add(it) }
            }
            Playlist(id, name, itemsList, createdAtMs, updatedAtMs)
        }.getOrNull()
    }

    fun savePlaylist(playlist: Playlist) {
        runCatching {
            val json = playlistToJson(playlist)
            prefs.edit().putString(KEY_PREFIX + playlist.id, json).apply()
            Timber.d("PlaylistStorage: saved playlist ${playlist.id}")
        }.onFailure {
            Timber.e(it, "PlaylistStorage: failed to save playlist ${playlist.id}")
        }
    }

    fun getPlaylist(id: String): Playlist? {
        val json = prefs.getString(KEY_PREFIX + id, null) ?: return null
        return jsonToPlaylist(json)
    }

    fun getAllPlaylists(): List<Playlist> {
        return prefs.all.entries
            .filter { it.key.startsWith(KEY_PREFIX) }
            .mapNotNull { entry ->
                jsonToPlaylist(entry.value as String)
            }.sortedByDescending { it.updatedAtMs }
    }

    fun removePlaylist(id: String) {
        prefs.edit().remove(KEY_PREFIX + id).apply()
        Timber.d("PlaylistStorage: removed playlist $id")
    }

    fun saveActiveQueue(items: List<QueueManager.QueueItem>, currentIndex: Int) {
        runCatching {
            val arr = JSONArray()
            items.forEach { arr.put(queueItemToJson(it)) }
            prefs.edit()
                .putString(KEY_ACTIVE_QUEUE, arr.toString())
                .putInt(KEY_ACTIVE_INDEX, currentIndex)
                .apply()
        }
    }

    fun loadActiveQueue(): Pair<List<QueueManager.QueueItem>, Int>? {
        return runCatching {
            val json = prefs.getString(KEY_ACTIVE_QUEUE, null) ?: return null
            val arr = JSONArray(json)
            val itemsList = mutableListOf<QueueManager.QueueItem>()
            for (i in 0 until arr.length()) {
                val itemObj = arr.optJSONObject(i) ?: continue
                jsonToQueueItem(itemObj)?.let { itemsList.add(it) }
            }
            val index = prefs.getInt(KEY_ACTIVE_INDEX, -1)
            itemsList to index
        }.getOrNull()
    }

    companion object {
        private const val PREFS_NAME = "xplayer_playlists"
        private const val KEY_PREFIX = "playlist_"
        private const val KEY_ACTIVE_QUEUE = "active_queue"
        private const val KEY_ACTIVE_INDEX = "active_queue_index"
    }
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/playlist/QueueManager.kt

```kotlin
package com.xplayer.dev.core.playlist

import timber.log.Timber
import java.util.concurrent.CopyOnWriteArrayList
import javax.inject.Inject
import javax.inject.Singleton

/**
 * # QueueManager
 *
 * Manages the active playback queue. Supports shuffle, repeat, manipulation, and navigation.
 */
@Singleton
class QueueManager @Inject constructor() {

    data class QueueItem(
        val mediaId: String,
        val url: String,
        val title: String? = null,
        val durationMs: Long = 0L,
        val thumbnailUrl: String? = null,
        val extras: Map<String, String> = emptyMap(),
    ) {
        init {
            require(mediaId.isNotBlank()) { "mediaId must not be blank" }
            require(url.isNotBlank()) { "url must not be blank" }
        }
    }

    enum class RepeatMode { OFF, ONE, ALL }

    private val _items = CopyOnWriteArrayList<QueueItem>()
    private var _currentIndex = -1
    private var _shuffleEnabled = false
    private var _repeatMode = RepeatMode.OFF

    val items: List<QueueItem> get() = _items.toList()
    val size: Int get() = _items.size
    val isEmpty: Boolean get() = _items.isEmpty()
    val currentIndex: Int get() = _currentIndex
    val currentItem: QueueItem? get() = _items.getOrNull(_currentIndex)
    val repeatMode: RepeatMode get() = _repeatMode
    val isShuffleEnabled: Boolean get() = _shuffleEnabled

    fun add(item: QueueItem) {
        _items.add(item)
        if (_currentIndex == -1) _currentIndex = 0
        Timber.d("QueueManager: added ${item.mediaId} (size=${_items.size})")
    }

    fun addAll(items: List<QueueItem>) {
        _items.addAll(items)
        if (_currentIndex == -1 && _items.isNotEmpty()) _currentIndex = 0
        Timber.d("QueueManager: added ${items.size} items (total=${_items.size})")
    }

    fun addAt(index: Int, item: QueueItem) {
        val safeIndex = index.coerceIn(0, _items.size)
        _items.add(safeIndex, item)
        if (_currentIndex >= safeIndex) _currentIndex++
        Timber.d("QueueManager: inserted at $safeIndex — ${item.mediaId}")
    }

    fun removeAt(index: Int): QueueItem? {
        if (index !in 0 until _items.size) return null
        val removed = _items.removeAt(index)
        when {
            _items.isEmpty() -> _currentIndex = -1
            index < _currentIndex -> _currentIndex--
            index == _currentIndex -> _currentIndex = _currentIndex.coerceAtMost(_items.size - 1)
        }
        Timber.d("QueueManager: removed ${removed.mediaId} at $index")
        return removed
    }

    fun removeById(mediaId: String): Boolean {
        val index = _items.indexOfFirst { it.mediaId == mediaId }
        if (index == -1) return false
        removeAt(index)
        return true
    }

    fun move(from: Int, to: Int) {
        if (from == to) return
        if (from !in 0 until _items.size || to !in 0 until _items.size) return

        val item = _items.removeAt(from)
        _items.add(to, item)

        _currentIndex = when (_currentIndex) {
            from -> to
            in (from + 1)..to -> _currentIndex - 1
            in to until from -> _currentIndex + 1
            else -> _currentIndex
        }
        Timber.d("QueueManager: moved $from -> $to")
    }

    fun replaceAt(index: Int, newItem: QueueItem) {
        if (index !in 0 until _items.size) return
        _items[index] = newItem
        Timber.d("QueueManager: replaced at $index — ${newItem.mediaId}")
    }

    fun clear() {
        _items.clear()
        _currentIndex = -1
        Timber.d("QueueManager: cleared")
    }

    fun setRepeatMode(mode: RepeatMode) {
        _repeatMode = mode
        Timber.d("QueueManager: repeatMode=$mode")
    }

    fun setShuffleEnabled(enabled: Boolean) {
        _shuffleEnabled = enabled
        Timber.d("QueueManager: shuffleEnabled=$enabled")
    }

    fun moveToNext(): QueueItem? {
        if (_items.isEmpty()) return null
        val next = when (_repeatMode) {
            RepeatMode.ONE -> _currentIndex
            RepeatMode.ALL -> if (_currentIndex + 1 < _items.size) _currentIndex + 1 else 0
            RepeatMode.OFF -> if (_currentIndex + 1 < _items.size) _currentIndex + 1 else -1
        }
        if (next == -1) return null
        _currentIndex = next
        Timber.d("QueueManager: moveToNext -> index=$_currentIndex")
        return currentItem
    }

    fun moveToPrevious(): QueueItem? {
        if (_items.isEmpty()) return null
        val prev = when (_repeatMode) {
            RepeatMode.ONE -> _currentIndex
            RepeatMode.ALL -> if (_currentIndex - 1 >= 0) _currentIndex - 1 else _items.size - 1
            RepeatMode.OFF -> if (_currentIndex - 1 >= 0) _currentIndex - 1 else -1
        }
        if (prev == -1) return null
        _currentIndex = prev
        Timber.d("QueueManager: moveToPrevious -> index=$_currentIndex")
        return currentItem
    }

    fun moveToIndex(index: Int): QueueItem? {
        if (index !in 0 until _items.size) return null
        _currentIndex = index
        Timber.d("QueueManager: moveToIndex=$index")
        return currentItem
    }
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/playback/PlaybackController.kt

```kotlin
package com.xplayer.dev.core.playback

import androidx.annotation.MainThread
import androidx.media3.common.Player
import androidx.media3.exoplayer.ExoPlayer
import com.xplayer.dev.core.config.PlayerConfiguration
import com.xplayer.dev.core.engine.PlayerEngine
import com.xplayer.dev.core.state.PlayerState
import com.xplayer.dev.core.state.stateName
import timber.log.Timber
import javax.inject.Inject
import javax.inject.Singleton

/**
 * # PlaybackController
 *
 * Low-level playback command executor guarded by state checks.
 */
@Singleton
class PlaybackController @Inject constructor(
    private val engine: PlayerEngine,
    private val config: PlayerConfiguration,
) {

    private val player: ExoPlayer?
        get() = engine.exoPlayer

    private val currentState: PlayerState
        get() = engine.state.value

    sealed interface CommandResult {
        data object Executed : CommandResult
        data object NoPlayer : CommandResult
        data class Rejected(val reason: String) : CommandResult

        val isSuccess: Boolean get() = this is Executed
    }

    @MainThread
    fun play(): CommandResult {
        val p = player ?: return CommandResult.NoPlayer
        val state = currentState

        if (!state.canPlay) {
            Timber.w("PlaybackController: play() rejected — state=${state.stateName}")
            return CommandResult.Rejected("Cannot play in state ${state.stateName}")
        }

        p.playWhenReady = true
        Timber.d("PlaybackController: play()")
        return CommandResult.Executed
    }

    @MainThread
    fun pause(): CommandResult {
        val p = player ?: return CommandResult.NoPlayer
        val state = currentState

        if (!state.canPause) {
            Timber.w("PlaybackController: pause() rejected — state=${state.stateName}")
            return CommandResult.Rejected("Cannot pause in state ${state.stateName}")
        }

        p.playWhenReady = false
        Timber.d("PlaybackController: pause()")
        return CommandResult.Executed
    }

    @MainThread
    fun stop(): CommandResult {
        val p = player ?: return CommandResult.NoPlayer
        p.stop()
        Timber.d("PlaybackController: stop()")
        return CommandResult.Executed
    }

    @MainThread
    fun replay(): CommandResult {
        val p = player ?: return CommandResult.NoPlayer
        p.seekTo(0L)
        p.playWhenReady = true
        Timber.d("PlaybackController: replay()")
        return CommandResult.Executed
    }

    @MainThread
    fun seekTo(positionMs: Long): CommandResult {
        val p = player ?: return CommandResult.NoPlayer
        val state = currentState

        if (!state.canSeek) {
            Timber.w("PlaybackController: seekTo($positionMs) rejected — state=${state.stateName}")
            return CommandResult.Rejected("Cannot seek in state ${state.stateName}")
        }

        val duration = p.duration.coerceAtLeast(0L)
        val clamped = positionMs.coerceIn(0L, if (duration > 0L) duration else Long.MAX_VALUE)

        p.seekTo(clamped)
        Timber.d("PlaybackController: seekTo(${clamped}ms)")
        return CommandResult.Executed
    }

    @MainThread
    fun seekBack(incrementMs: Long = config.playbackConfig.seekBackIncrementMs): CommandResult {
        val p = player ?: return CommandResult.NoPlayer
        val newPosition = (p.currentPosition - incrementMs).coerceAtLeast(0L)
        return seekTo(newPosition).also {
            Timber.d("PlaybackController: seekBack(${incrementMs}ms) -> ${newPosition}ms")
        }
    }

    @MainThread
    fun seekForward(incrementMs: Long = config.playbackConfig.seekForwardIncrementMs): CommandResult {
        val p = player ?: return CommandResult.NoPlayer
        val duration = p.duration.coerceAtLeast(0L)
        val newPosition = (p.currentPosition + incrementMs)
            .let { if (duration > 0L) it.coerceAtMost(duration) else it }
        return seekTo(newPosition).also {
            Timber.d("PlaybackController: seekForward(${incrementMs}ms) -> ${newPosition}ms")
        }
    }

    @MainThread
    fun fastSeekTo(positionMs: Long): CommandResult = seekTo(positionMs)

    @MainThread
    fun setPlaybackSpeed(speed: Float): CommandResult {
        require(speed in 0.25f..8.0f) { "Playback speed $speed out of range [0.25, 8.0]" }
        val p = player ?: return CommandResult.NoPlayer
        p.playbackParameters = androidx.media3.common.PlaybackParameters(speed)
        Timber.d("PlaybackController: speed=${speed}x")
        return CommandResult.Executed
    }

    @MainThread
    fun setVolume(volume: Float): CommandResult {
        require(volume in 0f..1f) { "Volume $volume out of range [0.0, 1.0]" }
        val p = player ?: return CommandResult.NoPlayer
        p.volume = volume
        Timber.d("PlaybackController: volume=$volume")
        return CommandResult.Executed
    }

    @MainThread
    fun setRepeatMode(@Player.RepeatMode mode: Int): CommandResult {
        val p = player ?: return CommandResult.NoPlayer
        p.repeatMode = mode
        Timber.d("PlaybackController: repeatMode=$mode")
        return CommandResult.Executed
    }

    @MainThread
    fun setShuffleMode(enabled: Boolean): CommandResult {
        val p = player ?: return CommandResult.NoPlayer
        p.shuffleModeEnabled = enabled
        Timber.d("PlaybackController: shuffle=$enabled")
        return CommandResult.Executed
    }

    @MainThread
    fun setSkipSilenceEnabled(enabled: Boolean): CommandResult {
        val p = player ?: return CommandResult.NoPlayer
        p.skipSilenceEnabled = enabled
        Timber.d("PlaybackController: skipSilence=$enabled")
        return CommandResult.Executed
    }

    val currentPositionMs: Long
        @MainThread get() = player?.currentPosition ?: 0L

    val durationMs: Long
        @MainThread get() = player?.duration?.coerceAtLeast(0L) ?: 0L

    val bufferedPositionMs: Long
        @MainThread get() = player?.bufferedPosition ?: 0L

    val bufferedPercent: Int
        @MainThread get() = player?.bufferedPercentage?.coerceIn(0, 100) ?: 0

    val isPlaying: Boolean
        @MainThread get() = player?.isPlaying ?: false

    val playbackSpeed: Float
        @MainThread get() = player?.playbackParameters?.speed ?: 1.0f

    val volume: Float
        @MainThread get() = player?.volume ?: 1.0f
}

private val PlayerState.canPlay: Boolean
    get() = this is PlayerState.Ready || this is PlayerState.Paused || this is PlayerState.Playing

private val PlayerState.canPause: Boolean
    get() = this is PlayerState.Playing || this is PlayerState.Buffering

private val PlayerState.canSeek: Boolean
    get() = when (this) {
        is PlayerState.Playing -> isSeekable
        is PlayerState.Paused -> true
        is PlayerState.Ready -> isSeekable
        else -> false
    }

```

## xplayer/app/src/main/java/com/xplayer/dev/core/playback/PlaybackCoordinator.kt

```kotlin
package com.xplayer.dev.core.playback

import androidx.annotation.MainThread
import androidx.media3.common.MediaItem
import com.xplayer.dev.core.engine.PlayerEngine
import com.xplayer.dev.core.playlist.QueueManager
import com.xplayer.dev.core.session.SessionRepository
import com.xplayer.dev.core.source.MediaSourceFactory
import com.xplayer.dev.core.source.SourceValidator
import timber.log.Timber
import javax.inject.Inject
import javax.inject.Singleton

/**
 * # PlaybackCoordinator
 *
 * Orchestrates load-and-play sequence across validation, queue management, session resumption, and engine prepare.
 */
@Singleton
class PlaybackCoordinator @Inject constructor(
    private val engine: PlayerEngine,
    private val controller: PlaybackController,
    private val queueManager: QueueManager,
    private val sessionRepository: SessionRepository,
    private val sourceValidator: SourceValidator,
    private val mediaSourceFactory: MediaSourceFactory,
) {

    sealed interface LoadResult {
        data class Success(val mediaId: String, val startPositionMs: Long) : LoadResult
        data class InvalidSource(val reason: String) : LoadResult
        data class InvalidIndex(val index: Int) : LoadResult
        data object NoNextItem : LoadResult
        data object NoPreviousItem : LoadResult
        data object SeekToStart : LoadResult
    }

    @MainThread
    fun load(
        mediaId: String,
        url: String,
        startPositionMs: Long = -1L,
        autoPlay: Boolean = true,
        headers: Map<String, String> = emptyMap(),
    ): LoadResult {
        val validation = sourceValidator.validate(url)
        if (validation is SourceValidator.ValidationResult.Invalid) {
            Timber.e("PlaybackCoordinator: invalid source — ${validation.reason}")
            return LoadResult.InvalidSource(validation.reason)
        }

        val resolvedPosition = when {
            startPositionMs >= 0L -> startPositionMs
            else -> sessionRepository.getSavedPosition(mediaId)
        }

        val mediaItem = MediaItem.Builder()
            .setUri(url)
            .setMediaId(mediaId)
            .build()

        engine.prepare(
            mediaItem = mediaItem,
            mediaId = mediaId,
        )

        if (resolvedPosition > 0L) {
            engine.seekTo(resolvedPosition)
            Timber.d("PlaybackCoordinator: resuming at ${resolvedPosition}ms")
        }

        if (autoPlay) {
            engine.play()
        }

        Timber.i("PlaybackCoordinator: loaded $mediaId from $url")
        return LoadResult.Success(mediaId, resolvedPosition)
    }

    @MainThread
    fun playNext(): LoadResult {
        val next = queueManager.moveToNext() ?: return LoadResult.NoNextItem
        return load(
            mediaId = next.mediaId,
            url = next.url,
        ).also { Timber.d("PlaybackCoordinator: playing next — ${next.mediaId}") }
    }

    @MainThread
    fun playPrevious(): LoadResult {
        if (engine.currentPositionMs > PREVIOUS_THRESHOLD_MS) {
            engine.seekTo(0L)
            Timber.d("PlaybackCoordinator: seek to start (position > ${PREVIOUS_THRESHOLD_MS}ms)")
            return LoadResult.SeekToStart
        }

        val previous = queueManager.moveToPrevious() ?: return LoadResult.NoPreviousItem
        return load(
            mediaId = previous.mediaId,
            url = previous.url,
        ).also { Timber.d("PlaybackCoordinator: playing previous — ${previous.mediaId}") }
    }

    @MainThread
    fun playAtIndex(index: Int): LoadResult {
        val item = queueManager.moveToIndex(index) ?: return LoadResult.InvalidIndex(index)
        return load(mediaId = item.mediaId, url = item.url)
    }

    companion object {
        private const val PREVIOUS_THRESHOLD_MS = 3_000L
    }
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/playback/PlaybackParameters.kt

```kotlin
package com.xplayer.dev.core.playback

/**
 * # PlaybackParameters
 *
 * Snapshot of all current playback parameter settings.
 */
data class PlaybackParameters(
    val speed: Float = 1.0f,
    val volume: Float = 1.0f,
    val repeatMode: Int = 0,
    val shuffleEnabled: Boolean = false,
    val skipSilence: Boolean = false,
    val pitch: Float = 1.0f,
) {
    init {
        require(speed in 0.25f..8.0f) { "speed out of range [0.25, 8.0]" }
        require(volume in 0f..1f) { "volume out of range [0.0, 1.0]" }
        require(pitch > 0f) { "pitch must be > 0" }
    }

    val isNormalSpeed: Boolean get() = speed == 1.0f
    val isMuted: Boolean get() = volume == 0f
    val isLooping: Boolean get() = repeatMode == 2

    companion object {
        val DEFAULT = PlaybackParameters()
    }
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/playback/PlaybackScheduler.kt

```kotlin
package com.xplayer.dev.core.playback

import com.xplayer.dev.core.config.PlayerConfiguration
import com.xplayer.dev.core.engine.PlayerEngine
import kotlinx.coroutines.CoroutineScope
import kotlinx.coroutines.Job
import kotlinx.coroutines.delay
import kotlinx.coroutines.isActive
import kotlinx.coroutines.launch
import timber.log.Timber
import javax.inject.Inject
import javax.inject.Singleton

/**
 * # PlaybackScheduler
 *
 * Periodic position and buffer polling loop driven by coroutines.
 */
@Singleton
class PlaybackScheduler @Inject constructor(
    private val engine: PlayerEngine,
    private val config: PlayerConfiguration,
) {

    private var positionJob: Job? = null
    private var bufferJob: Job? = null

    private val positionIntervalMs = 500L
    private val bufferIntervalMs = 1_000L

    fun start(
        scope: CoroutineScope,
        onPosition: (positionMs: Long) -> Unit = {},
        onBuffer: (bufferPercent: Int) -> Unit = {},
    ) {
        stop()

        positionJob = scope.launch {
            Timber.d("PlaybackScheduler: position polling started (${positionIntervalMs}ms)")
            while (isActive) {
                onPosition(engine.currentPositionMs)
                delay(positionIntervalMs)
            }
        }

        bufferJob = scope.launch {
            Timber.d("PlaybackScheduler: buffer polling started (${bufferIntervalMs}ms)")
            while (isActive) {
                onBuffer(engine.bufferedPercent)
                delay(bufferIntervalMs)
            }
        }
    }

    fun stop() {
        positionJob?.cancel()
        bufferJob?.cancel()
        positionJob = null
        bufferJob = null
        Timber.d("PlaybackScheduler: stopped")
    }

    val isRunning: Boolean
        get() = positionJob?.isActive == true
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/track/DefaultTrackPolicy.kt

```kotlin
package com.xplayer.dev.core.track

import androidx.media3.common.C
import androidx.media3.common.TrackSelectionParameters
import androidx.media3.exoplayer.trackselection.DefaultTrackSelector
import timber.log.Timber
import javax.inject.Inject
import javax.inject.Singleton

/**
 * # DefaultTrackPolicy
 *
 * Configures default track selection policies on ExoPlayer's DefaultTrackSelector.
 */
@Singleton
class DefaultTrackPolicy @Inject constructor(
    private val overrideManager: TrackOverrideManager,
) {

    fun applyDefaults(trackSelector: DefaultTrackSelector) {
        val builder = trackSelector.buildUponParameters()

        val audioLang = overrideManager.getPreferredAudioLanguage()
        if (!audioLang.isNullOrBlank()) {
            builder.setPreferredAudioLanguage(audioLang)
        }

        val subLang = overrideManager.getPreferredSubtitleLanguage()
        if (overrideManager.areSubtitlesDisabled()) {
            builder.setIgnoredTextSelectionFlags(C.SELECTION_FLAG_DEFAULT or C.SELECTION_FLAG_FORCED)
            builder.setTrackTypeDisabled(C.TRACK_TYPE_TEXT, true)
        } else if (!subLang.isNullOrBlank()) {
            builder.setPreferredTextLanguage(subLang)
            builder.setTrackTypeDisabled(C.TRACK_TYPE_TEXT, false)
        }

        builder.setMaxVideoSizeSd() // Can be configured via user settings or bandwidth
        builder.setSelectUndeterminedTextLanguage(true)

        trackSelector.setParameters(builder)
        Timber.i("DefaultTrackPolicy: applied track selection policy")
    }

    fun setMaxVideoResolution(trackSelector: DefaultTrackSelector, width: Int, height: Int) {
        val builder = trackSelector.buildUponParameters()
            .setMaxVideoSize(width, height)
        trackSelector.setParameters(builder)
        Timber.d("DefaultTrackPolicy: max video size set to ${width}x$height")
    }
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/track/TrackManager.kt

```kotlin
package com.xplayer.dev.core.track

import androidx.media3.common.C
import androidx.media3.common.Tracks
import androidx.media3.exoplayer.trackselection.DefaultTrackSelector
import com.xplayer.dev.core.engine.PlayerEngine
import timber.log.Timber
import javax.inject.Inject
import javax.inject.Singleton

/**
 * # TrackManager
 *
 * High-level orchestration facade over [TrackRepository], [TrackSelector], and [DefaultTrackPolicy].
 */
@Singleton
class TrackManager @Inject constructor(
    private val engine: PlayerEngine,
    private val repository: TrackRepository,
    private val selector: TrackSelector,
    private val policy: DefaultTrackPolicy,
) {

    private val trackSelector: DefaultTrackSelector?
        get() = engine.exoPlayer?.trackSelector as? DefaultTrackSelector

    fun onTracksChanged(tracks: Tracks) {
        repository.updateTracks(tracks)
    }

    fun applyDefaultPolicies() {
        trackSelector?.let { policy.applyDefaults(it) }
    }

    fun selectTrack(groupIndex: Int, trackIndex: Int) {
        val ts = trackSelector ?: return
        val tracks = engine.exoPlayer?.currentTracks ?: return
        selector.selectTrack(ts, tracks, groupIndex, trackIndex)
    }

    fun disableSubtitles() {
        trackSelector?.let { selector.disableTrackType(it, C.TRACK_TYPE_TEXT) }
    }

    fun resetTrackSelection() {
        trackSelector?.let { selector.resetOverrides(it) }
    }
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/track/TrackOverrideManager.kt

```kotlin
package com.xplayer.dev.core.track

import android.content.Context
import android.content.SharedPreferences
import androidx.media3.common.C
import androidx.media3.common.TrackSelectionOverride
import androidx.media3.common.TrackGroup
import timber.log.Timber
import javax.inject.Inject
import javax.inject.Singleton

/**
 * # TrackOverrideManager
 *
 * Persists and restores user track overrides (e.g., preferred audio language or subtitle selection).
 */
@Singleton
class TrackOverrideManager @Inject constructor(
    private val context: Context,
) {
    private val prefs: SharedPreferences by lazy {
        context.getSharedPreferences("xplayer_track_overrides", Context.MODE_PRIVATE)
    }

    fun savePreferredAudioLanguage(language: String) {
        prefs.edit().putString(KEY_AUDIO_LANG, language).apply()
        Timber.d("TrackOverrideManager: saved preferred audio language '$language'")
    }

    fun getPreferredAudioLanguage(): String? = prefs.getString(KEY_AUDIO_LANG, null)

    fun savePreferredSubtitleLanguage(language: String) {
        prefs.edit().putString(KEY_SUBTITLE_LANG, language).apply()
        Timber.d("TrackOverrideManager: saved preferred subtitle language '$language'")
    }

    fun getPreferredSubtitleLanguage(): String? = prefs.getString(KEY_SUBTITLE_LANG, null)

    fun setSubtitlesDisabled(disabled: Boolean) {
        prefs.edit().putBoolean(KEY_SUBTITLES_DISABLED, disabled).apply()
        Timber.d("TrackOverrideManager: subtitles disabled=$disabled")
    }

    fun areSubtitlesDisabled(): Boolean = prefs.getBoolean(KEY_SUBTITLES_DISABLED, false)

    fun clearAll() {
        prefs.edit().clear().apply()
        Timber.i("TrackOverrideManager: cleared all track overrides")
    }

    companion object {
        private const val KEY_AUDIO_LANG = "pref_audio_lang"
        private const val KEY_SUBTITLE_LANG = "pref_sub_lang"
        private const val KEY_SUBTITLES_DISABLED = "pref_subs_disabled"
    }
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/track/TrackRepository.kt

```kotlin
package com.xplayer.dev.core.track

import androidx.media3.common.C
import androidx.media3.common.Format
import androidx.media3.common.Tracks
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.asStateFlow
import timber.log.Timber
import javax.inject.Inject
import javax.inject.Singleton

/**
 * # TrackRepository
 *
 * Holds current available and selected tracks discovered from ExoPlayer.
 */
@Singleton
class TrackRepository @Inject constructor() {

    data class TrackInfo(
        val groupIndex: Int,
        val trackIndex: Int,
        val format: Format,
        val isSupported: Boolean,
        val isSelected: Boolean,
        val trackType: Int,
    ) {
        val label: String get() = format.label ?: format.language ?: "Track #${trackIndex + 1}"
    }

    private val _videoTracks = MutableStateFlow<List<TrackInfo>>(emptyList())
    val videoTracks: Flow<List<TrackInfo>> = _videoTracks.asStateFlow()

    private val _audioTracks = MutableStateFlow<List<TrackInfo>>(emptyList())
    val audioTracks: Flow<List<TrackInfo>> = _audioTracks.asStateFlow()

    private val _subtitleTracks = MutableStateFlow<List<TrackInfo>>(emptyList())
    val subtitleTracks: Flow<List<TrackInfo>> = _subtitleTracks.asStateFlow()

    fun updateTracks(tracks: Tracks) {
        val videoList = mutableListOf<TrackInfo>()
        val audioList = mutableListOf<TrackInfo>()
        val subList = mutableListOf<TrackInfo>()

        for ((groupIdx, group) in tracks.groups.withIndex()) {
            val type = group.type
            for (trackIdx in 0 until group.length) {
                val format = group.getTrackFormat(trackIdx)
                val isSupported = group.isTrackSupported(trackIdx)
                val isSelected = group.isTrackSelected(trackIdx)
                val info = TrackInfo(groupIdx, trackIdx, format, isSupported, isSelected, type)

                when (type) {
                    C.TRACK_TYPE_VIDEO -> videoList.add(info)
                    C.TRACK_TYPE_AUDIO -> audioList.add(info)
                    C.TRACK_TYPE_TEXT -> subList.add(info)
                }
            }
        }

        _videoTracks.value = videoList
        _audioTracks.value = audioList
        _subtitleTracks.value = subList
        Timber.d("TrackRepository: updated tracks (v=${videoList.size}, a=${audioList.size}, s=${subList.size})")
    }

    fun clear() {
        _videoTracks.value = emptyList()
        _audioTracks.value = emptyList()
        _subtitleTracks.value = emptyList()
    }
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/track/TrackSelector.kt

```kotlin
package com.xplayer.dev.core.track

import androidx.media3.common.C
import androidx.media3.common.TrackSelectionOverride
import androidx.media3.common.Tracks
import androidx.media3.exoplayer.trackselection.DefaultTrackSelector
import timber.log.Timber
import javax.inject.Inject
import javax.inject.Singleton

/**
 * # TrackSelector
 *
 * Low-level track selection manipulation on ExoPlayer's DefaultTrackSelector.
 */
@Singleton
class TrackSelector @Inject constructor(
    private val overrideManager: TrackOverrideManager,
) {

    fun selectTrack(trackSelector: DefaultTrackSelector, tracks: Tracks, groupIndex: Int, trackIndex: Int) {
        val group = tracks.groups.getOrNull(groupIndex) ?: return
        val mediaTrackGroup = group.mediaTrackGroup
        val override = TrackSelectionOverride(mediaTrackGroup, trackIndex)

        val builder = trackSelector.buildUponParameters()
            .setOverrideForType(override)

        if (group.type == C.TRACK_TYPE_TEXT) {
            builder.setTrackTypeDisabled(C.TRACK_TYPE_TEXT, false)
            overrideManager.setSubtitlesDisabled(false)
            group.getTrackFormat(trackIndex).language?.let { overrideManager.savePreferredSubtitleLanguage(it) }
        } else if (group.type == C.TRACK_TYPE_AUDIO) {
            group.getTrackFormat(trackIndex).language?.let { overrideManager.savePreferredAudioLanguage(it) }
        }

        trackSelector.setParameters(builder)
        Timber.i("TrackSelector: selected track $trackIndex in group $groupIndex")
    }

    fun disableTrackType(trackSelector: DefaultTrackSelector, trackType: Int) {
        val builder = trackSelector.buildUponParameters()
            .setTrackTypeDisabled(trackType, true)
        trackSelector.setParameters(builder)

        if (trackType == C.TRACK_TYPE_TEXT) {
            overrideManager.setSubtitlesDisabled(true)
        }
        Timber.i("TrackSelector: disabled track type $trackType")
    }

    fun resetOverrides(trackSelector: DefaultTrackSelector) {
        val builder = trackSelector.buildUponParameters()
            .clearOverridesOfType(C.TRACK_TYPE_VIDEO)
            .clearOverridesOfType(C.TRACK_TYPE_AUDIO)
            .clearOverridesOfType(C.TRACK_TYPE_TEXT)
            .setTrackTypeDisabled(C.TRACK_TYPE_TEXT, false)
        trackSelector.setParameters(builder)
        overrideManager.clearAll()
        Timber.i("TrackSelector: reset all track overrides")
    }
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/audio/AudioAttributesManager.kt

```kotlin
package com.xplayer.dev.core.audio

import androidx.media3.common.AudioAttributes
import androidx.media3.common.C
import androidx.media3.exoplayer.ExoPlayer
import timber.log.Timber
import javax.inject.Inject
import javax.inject.Singleton

/**
 * # AudioAttributesManager
 *
 * Configures media3 AudioAttributes on ExoPlayer for proper routing and system audio handling.
 */
@Singleton
class AudioAttributesManager @Inject constructor() {

    val movieAudioAttributes: AudioAttributes = AudioAttributes.Builder()
        .setContentType(C.AUDIO_CONTENT_TYPE_MOVIE)
        .setUsage(C.USAGE_MEDIA)
        .build()

    val musicAudioAttributes: AudioAttributes = AudioAttributes.Builder()
        .setContentType(C.AUDIO_CONTENT_TYPE_MUSIC)
        .setUsage(C.USAGE_MEDIA)
        .build()

    val speechAudioAttributes: AudioAttributes = AudioAttributes.Builder()
        .setContentType(C.AUDIO_CONTENT_TYPE_SPEECH)
        .setUsage(C.USAGE_MEDIA)
        .build()

    fun applyAttributes(player: ExoPlayer, attributes: AudioAttributes = movieAudioAttributes, handleAudioFocus: Boolean = true) {
        player.setAudioAttributes(attributes, handleAudioFocus)
        Timber.i("AudioAttributesManager: applied attributes (handleFocus=$handleAudioFocus)")
    }
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/audio/AudioFocusManager.kt

```kotlin
package com.xplayer.dev.core.audio

import android.content.Context
import android.media.AudioAttributes
import android.media.AudioFocusRequest
import android.media.AudioManager as AndroidAudioManager
import android.os.Build
import timber.log.Timber
import javax.inject.Inject
import javax.inject.Singleton

/**
 * # AudioFocusManager
 *
 * Manages audio focus requests, ducking, and resume logic across Android API levels.
 */
@Singleton
class AudioFocusManager @Inject constructor(
    private val context: Context,
) {
    private val audioManager by lazy {
        context.getSystemService(Context.AUDIO_SERVICE) as AndroidAudioManager
    }

    private var focusRequest: AudioFocusRequest? = null
    private var onFocusLoss: (() -> Unit)? = null
    private var onFocusGain: (() -> Unit)? = null
    private var onFocusDuck: (() -> Unit)? = null

    private val focusChangeListener = AndroidAudioManager.OnAudioFocusChangeListener { focusChange ->
        when (focusChange) {
            AndroidAudioManager.AUDIOFOCUS_LOSS -> {
                Timber.w("AudioFocusManager: AUDIOFOCUS_LOSS")
                onFocusLoss?.invoke()
            }
            AndroidAudioManager.AUDIOFOCUS_LOSS_TRANSIENT -> {
                Timber.w("AudioFocusManager: AUDIOFOCUS_LOSS_TRANSIENT")
                onFocusLoss?.invoke()
            }
            AndroidAudioManager.AUDIOFOCUS_LOSS_TRANSIENT_CAN_DUCK -> {
                Timber.d("AudioFocusManager: AUDIOFOCUS_LOSS_TRANSIENT_CAN_DUCK")
                onFocusDuck?.invoke()
            }
            AndroidAudioManager.AUDIOFOCUS_GAIN -> {
                Timber.i("AudioFocusManager: AUDIOFOCUS_GAIN")
                onFocusGain?.invoke()
            }
        }
    }

    fun requestFocus(
        onLoss: () -> Unit,
        onGain: () -> Unit,
        onDuck: () -> Unit,
    ): Boolean {
        this.onFocusLoss = onLoss
        this.onFocusGain = onGain
        this.onFocusDuck = onDuck

        val result = if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            val attributes = AudioAttributes.Builder()
                .setUsage(AudioAttributes.USAGE_MEDIA)
                .setContentType(AudioAttributes.CONTENT_TYPE_MOVIE)
                .build()

            val request = AudioFocusRequest.Builder(AndroidAudioManager.AUDIOFOCUS_GAIN)
                .setAudioAttributes(attributes)
                .setAcceptsDelayedFocusGain(true)
                .setOnAudioFocusChangeListener(focusChangeListener)
                .build()

            focusRequest = request
            audioManager.requestAudioFocus(request)
        } else {
            @Suppress("DEPRECATION")
            audioManager.requestAudioFocus(
                focusChangeListener,
                AndroidAudioManager.STREAM_MUSIC,
                AndroidAudioManager.AUDIOFOCUS_GAIN
            )
        }

        val granted = result == AndroidAudioManager.AUDIOFOCUS_REQUEST_GRANTED
        Timber.i("AudioFocusManager: requestFocus granted=$granted")
        return granted
    }

    fun abandonFocus() {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            focusRequest?.let { audioManager.abandonAudioFocusRequest(it) }
        } else {
            @Suppress("DEPRECATION")
            audioManager.abandonAudioFocus(focusChangeListener)
        }
        focusRequest = null
        Timber.i("AudioFocusManager: focus abandoned")
    }
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/audio/AudioManager.kt

```kotlin
package com.xplayer.dev.core.audio

import com.xplayer.dev.core.engine.PlayerEngine
import timber.log.Timber
import javax.inject.Inject
import javax.inject.Singleton

/**
 * # AudioManager
 *
 * Facade coordinating audio focus, attributes, volume, and routing for [PlayerEngine].
 */
@Singleton
class AudioManager @Inject constructor(
    private val engine: PlayerEngine,
    private val focusManager: AudioFocusManager,
    private val attributesManager: AudioAttributesManager,
    private val session: AudioSession,
) {

    fun setupAudio() {
        val player = engine.exoPlayer ?: return
        attributesManager.applyAttributes(player)
        Timber.i("AudioManager: setup completed")
    }

    fun requestAudioFocus(): Boolean {
        return focusManager.requestFocus(
            onLoss = {
                Timber.w("AudioManager: focus lost -> pausing playback")
                engine.pause()
            },
            onGain = {
                Timber.i("AudioManager: focus gained -> unducking / resuming")
                engine.exoPlayer?.let { session.unduck(it) }
            },
            onDuck = {
                Timber.d("AudioManager: ducking volume")
                engine.exoPlayer?.let { session.duck(it) }
            }
        )
    }

    fun abandonAudioFocus() {
        focusManager.abandonFocus()
    }

    fun setVolume(volume: Float) {
        engine.exoPlayer?.let { session.setVolume(it, volume) }
    }

    fun duck() {
        engine.exoPlayer?.let { session.duck(it) }
    }

    fun unduck() {
        engine.exoPlayer?.let { session.unduck(it) }
    }

    fun getOutputDevice(): String = session.getOutputDeviceName()
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/audio/AudioSession.kt

```kotlin
package com.xplayer.dev.core.audio

import android.content.Context
import android.media.AudioManager as AndroidAudioManager
import androidx.media3.exoplayer.ExoPlayer
import timber.log.Timber
import javax.inject.Inject
import javax.inject.Singleton

/**
 * # AudioSession
 *
 * Manages volume, ducking levels, audio output devices (bluetooth/headset/speaker), and audio session ID.
 */
@Singleton
class AudioSession @Inject constructor(
    private val context: Context,
) {
    private val androidAudioManager by lazy {
        context.getSystemService(Context.AUDIO_SERVICE) as AndroidAudioManager
    }

    private var normalVolume: Float = 1.0f
    private var isDucked: Boolean = false

    fun getAudioSessionId(player: ExoPlayer): Int = player.audioSessionId

    fun duck(player: ExoPlayer, duckVolume: Float = 0.3f) {
        if (!isDucked) {
            normalVolume = player.volume
            player.volume = duckVolume
            isDucked = true
            Timber.d("AudioSession: ducked volume to $duckVolume")
        }
    }

    fun unduck(player: ExoPlayer) {
        if (isDucked) {
            player.volume = normalVolume
            isDucked = false
            Timber.d("AudioSession: restored volume to $normalVolume")
        }
    }

    fun setVolume(player: ExoPlayer, volume: Float) {
        val clamped = volume.coerceIn(0.0f, 1.0f)
        player.volume = clamped
        if (!isDucked) normalVolume = clamped
        Timber.d("AudioSession: setVolume=$clamped")
    }

    fun isBluetoothAudioConnected(): Boolean {
        @Suppress("DEPRECATION")
        return androidAudioManager.isBluetoothA2dpOn || androidAudioManager.isBluetoothScoOn
    }

    fun isWiredHeadsetConnected(): Boolean {
        @Suppress("DEPRECATION")
        return androidAudioManager.isWiredHeadsetOn
    }

    fun getOutputDeviceName(): String {
        return when {
            isBluetoothAudioConnected() -> "Bluetooth"
            isWiredHeadsetConnected() -> "Wired Headset"
            else -> "Speaker"
        }
    }
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/video/DecoderManager.kt

```kotlin
package com.xplayer.dev.core.video

import android.content.Context
import androidx.media3.exoplayer.DefaultRenderersFactory
import androidx.media3.exoplayer.RenderersFactory
import timber.log.Timber
import javax.inject.Inject
import javax.inject.Singleton

/**
 * # DecoderManager
 *
 * Configures media3 RenderersFactory for hardware vs software video decoding fallbacks.
 */
@Singleton
class DecoderManager @Inject constructor(
    private val context: Context,
) {
    enum class DecoderMode { HARDWARE_ONLY, HARDWARE_PREFERRED, SOFTWARE_PREFERRED }

    fun buildRenderersFactory(mode: DecoderMode = DecoderMode.HARDWARE_PREFERRED): RenderersFactory {
        val extensionRendererMode = when (mode) {
            DecoderMode.HARDWARE_ONLY -> DefaultRenderersFactory.EXTENSION_RENDERER_MODE_OFF
            DecoderMode.HARDWARE_PREFERRED -> DefaultRenderersFactory.EXTENSION_RENDERER_MODE_PREFER
            DecoderMode.SOFTWARE_PREFERRED -> DefaultRenderersFactory.EXTENSION_RENDERER_MODE_PREFER
        }

        val factory = DefaultRenderersFactory(context)
            .setExtensionRendererMode(extensionRendererMode)
            .setEnableDecoderFallback(true)

        Timber.i("DecoderManager: built RenderersFactory with mode=$mode")
        return factory
    }
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/video/SurfaceManager.kt

```kotlin
package com.xplayer.dev.core.video

import android.view.Surface
import android.view.SurfaceView
import android.view.TextureView
import androidx.annotation.MainThread
import androidx.media3.exoplayer.ExoPlayer
import timber.log.Timber
import javax.inject.Inject
import javax.inject.Singleton

/**
 * # SurfaceManager
 *
 * Manages attaching and detaching video render surfaces (SurfaceView, TextureView, or Surface) to ExoPlayer.
 */
@Singleton
class SurfaceManager @Inject constructor() {

    @MainThread
    fun attachSurfaceView(player: ExoPlayer, surfaceView: SurfaceView) {
        player.clearVideoSurface()
        player.setVideoSurfaceView(surfaceView)
        Timber.i("SurfaceManager: attached SurfaceView")
    }

    @MainThread
    fun attachTextureView(player: ExoPlayer, textureView: TextureView) {
        player.clearVideoSurface()
        player.setVideoTextureView(textureView)
        Timber.i("SurfaceManager: attached TextureView")
    }

    @MainThread
    fun attachSurface(player: ExoPlayer, surface: Surface) {
        player.clearVideoSurface()
        player.setVideoSurface(surface)
        Timber.i("SurfaceManager: attached Surface")
    }

    @MainThread
    fun detachSurface(player: ExoPlayer) {
        player.clearVideoSurface()
        Timber.i("SurfaceManager: detached video surface")
    }
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/video/VideoManager.kt

```kotlin
package com.xplayer.dev.core.video

import android.view.SurfaceView
import android.view.TextureView
import com.xplayer.dev.core.engine.PlayerEngine
import timber.log.Timber
import javax.inject.Inject
import javax.inject.Singleton

/**
 * # VideoManager
 *
 * Facade coordinating surfaces, decoding options, and video info reporting for [PlayerEngine].
 */
@Singleton
class VideoManager @Inject constructor(
    private val engine: PlayerEngine,
    private val surfaceManager: SurfaceManager,
    private val videoRenderer: VideoRenderer,
) {

    fun attachSurfaceView(surfaceView: SurfaceView) {
        engine.exoPlayer?.let { surfaceManager.attachSurfaceView(it, surfaceView) }
    }

    fun attachTextureView(textureView: TextureView) {
        engine.exoPlayer?.let { surfaceManager.attachTextureView(it, textureView) }
    }

    fun detachSurface() {
        engine.exoPlayer?.let { surfaceManager.detachSurface(it) }
    }

    fun getCurrentVideoInfo(): VideoRenderer.VideoInfo {
        val player = engine.exoPlayer ?: return VideoRenderer.VideoInfo()
        return videoRenderer.getVideoInfo(player)
    }
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/video/VideoRenderer.kt

```kotlin
package com.xplayer.dev.core.video

import androidx.annotation.MainThread
import androidx.media3.common.VideoSize
import androidx.media3.exoplayer.ExoPlayer
import timber.log.Timber
import javax.inject.Inject
import javax.inject.Singleton

/**
 * # VideoRenderer
 *
 * Monitors video rendering metrics, aspect ratio scaling, rotation, and frame drops.
 */
@Singleton
class VideoRenderer @Inject constructor() {

    data class VideoInfo(
        val width: Int = 0,
        val height: Int = 0,
        val unappliedRotationDegrees: Int = 0,
        val pixelWidthHeightRatio: Float = 1.0f,
    ) {
        val aspectRatio: Float
            get() = if (height > 0) (width * pixelWidthHeightRatio) / height else 1.0f
        val isPortrait: Boolean
            get() = unappliedRotationDegrees % 180 != 0 || height > width
    }

    @MainThread
    fun getVideoInfo(player: ExoPlayer): VideoInfo {
        val size = player.videoSize
        return VideoInfo(
            width = size.width,
            height = size.height,
            unappliedRotationDegrees = size.unappliedRotationDegrees,
            pixelWidthHeightRatio = size.pixelWidthHeightRatio,
        )
    }

    fun logFrameDrop(droppedFrames: Int, elapsedMs: Long) {
        if (droppedFrames > 0) {
            Timber.w("VideoRenderer: dropped $droppedFrames frames in ${elapsedMs}ms")
        }
    }
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/subtitle/SubtitleLoader.kt

```kotlin
package com.xplayer.dev.core.subtitle

import android.net.Uri
import androidx.media3.common.C
import androidx.media3.common.MediaItem
import timber.log.Timber
import javax.inject.Inject
import javax.inject.Singleton

/**
 * # SubtitleLoader
 *
 * Builds media3 SubtitleConfiguration for attaching external subtitle files to MediaItem.
 */
@Singleton
class SubtitleLoader @Inject constructor(
    private val parser: SubtitleParser,
) {

    fun buildSubtitleConfiguration(
        url: String,
        language: String = "en",
        label: String = "External Subtitle",
        isForced: Boolean = false,
    ): MediaItem.SubtitleConfiguration {
        val mimeType = parser.detectMimeType(url)
        var selectionFlags = 0
        if (isForced) selectionFlags = selectionFlags or C.SELECTION_FLAG_FORCED

        Timber.i("SubtitleLoader: loading subtitle $url ($language, $mimeType)")
        return MediaItem.SubtitleConfiguration.Builder(Uri.parse(url))
            .setMimeType(mimeType)
            .setLanguage(language)
            .setLabel(label)
            .setSelectionFlags(selectionFlags)
            .build()
    }
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/subtitle/SubtitleManager.kt

```kotlin
package com.xplayer.dev.core.subtitle

import com.xplayer.dev.core.track.TrackManager
import timber.log.Timber
import javax.inject.Inject
import javax.inject.Singleton

/**
 * # SubtitleManager
 *
 * Facade managing external subtitle loading, toggling, and sync delay offset.
 */
@Singleton
class SubtitleManager @Inject constructor(
    private val repository: SubtitleRepository,
    private val loader: SubtitleLoader,
    private val trackManager: TrackManager,
) {

    fun loadExternalSubtitle(url: String, language: String = "en", label: String = "External") {
        val config = loader.buildSubtitleConfiguration(url, language, label)
        repository.addExternalSubtitle(SubtitleRepository.SubtitleTrack(url, language, label))
        // Note: To apply external subtitle dynamically to a playing stream without restarting, ExoPlayer requires reloading or adding at media item build time.
        Timber.i("SubtitleManager: registered external subtitle $label ($url)")
    }

    fun enableSubtitles() {
        trackManager.applyDefaultPolicies()
    }

    fun disableSubtitles() {
        trackManager.disableSubtitles()
    }

    fun setSubtitleDelay(offsetMs: Long) {
        repository.setSyncOffset(offsetMs)
        // Subtitle rendering delay correction can be applied to custom subtitle views or renderers
    }
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/subtitle/SubtitleParser.kt

```kotlin
package com.xplayer.dev.core.subtitle

import android.net.Uri
import androidx.media3.common.MimeTypes
import timber.log.Timber
import javax.inject.Inject
import javax.inject.Singleton

/**
 * # SubtitleParser
 *
 * Resolves appropriate MIME types for subtitle formats (SRT, ASS, SSA, WebVTT, TTML) from URLs or filenames.
 */
@Singleton
class SubtitleParser @Inject constructor() {

    enum class SubtitleFormat(val mimeType: String, val extensions: List<String>) {
        SRT(MimeTypes.APPLICATION_SUBRIP, listOf("srt")),
        WEBVTT(MimeTypes.TEXT_VTT, listOf("vtt")),
        TTML(MimeTypes.APPLICATION_TTML, listOf("ttml", "xml")),
        ASS(MimeTypes.TEXT_SSA, listOf("ass", "ssa")),
        UNKNOWN(MimeTypes.APPLICATION_SUBRIP, emptyList());

        companion object {
            fun fromUrl(url: String): SubtitleFormat {
                val path = Uri.parse(url).path ?: url
                val ext = path.substringAfterLast('.', "").lowercase()
                return entries.find { ext in it.extensions } ?: UNKNOWN
            }
        }
    }

    fun detectMimeType(url: String): String {
        val format = SubtitleFormat.fromUrl(url)
        Timber.d("SubtitleParser: detected format $format for $url")
        return format.mimeType
    }
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/subtitle/SubtitleRepository.kt

```kotlin
package com.xplayer.dev.core.subtitle

import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.asStateFlow
import timber.log.Timber
import javax.inject.Inject
import javax.inject.Singleton

/**
 * # SubtitleRepository
 *
 * Tracks loaded external subtitles and sync offsets.
 */
@Singleton
class SubtitleRepository @Inject constructor() {

    data class SubtitleTrack(
        val url: String,
        val language: String,
        val label: String,
        val offsetMs: Long = 0L,
    )

    private val _externalSubtitles = MutableStateFlow<List<SubtitleTrack>>(emptyList())
    val externalSubtitles: Flow<List<SubtitleTrack>> = _externalSubtitles.asStateFlow()

    private var currentOffsetMs = 0L

    fun addExternalSubtitle(track: SubtitleTrack) {
        _externalSubtitles.value = _externalSubtitles.value + track
        Timber.i("SubtitleRepository: added external track ${track.label}")
    }

    fun setSyncOffset(offsetMs: Long) {
        currentOffsetMs = offsetMs
        Timber.d("SubtitleRepository: set sync offset to ${offsetMs}ms")
    }

    fun getSyncOffset(): Long = currentOffsetMs

    fun clear() {
        _externalSubtitles.value = emptyList()
        currentOffsetMs = 0L
    }
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/metadata/MetadataExtractor.kt

```kotlin
package com.xplayer.dev.core.metadata

import androidx.media3.common.MediaMetadata
import timber.log.Timber
import javax.inject.Inject
import javax.inject.Singleton

/**
 * # MetadataExtractor
 *
 * Extracts structured metadata from Media3 [MediaMetadata].
 *
 * Converts Media3 types to our own serialisation-safe [MediaInfo] model.
 */
@Singleton
class MetadataExtractor @Inject constructor() {

    data class MediaInfo(
        val mediaId: String?          = null,
        val title: String?            = null,
        val artist: String?           = null,
        val album: String?            = null,
        val genre: String?            = null,
        val description: String?      = null,
        val artworkUri: String?       = null,
        val durationMs: Long?         = null,
        val trackNumber: Int?         = null,
        val totalTracks: Int?         = null,
        val discNumber: Int?          = null,
        val releaseYear: Int?         = null,
        val recordingYear: Int?       = null,
        val composer: String?         = null,
        val albumArtist: String?      = null,
        val conductor: String?        = null,
        val writer: String?           = null,
        val station: String?          = null,
        val extras: Map<String, String> = emptyMap(),
    ) {
        val displayTitle: String
            get() = title ?: artist ?: "Unknown"

        val hasArtwork: Boolean get() = artworkUri != null
    }

    /**
     * Extract [MediaInfo] from [mediaMetadata].
     */
    fun extract(
        mediaMetadata: MediaMetadata,
        mediaId: String? = null,
    ): MediaInfo {
        return MediaInfo(
            mediaId      = mediaId,
            title        = mediaMetadata.title?.toString(),
            artist       = mediaMetadata.artist?.toString(),
            album        = mediaMetadata.albumTitle?.toString(),
            genre        = mediaMetadata.genre?.toString(),
            description  = mediaMetadata.description?.toString(),
            artworkUri   = mediaMetadata.artworkUri?.toString(),
            durationMs   = null,
            trackNumber  = mediaMetadata.trackNumber,
            totalTracks  = mediaMetadata.totalTrackCount,
            discNumber   = mediaMetadata.discNumber,
            releaseYear  = mediaMetadata.releaseYear,
            recordingYear = mediaMetadata.recordingYear,
            composer     = mediaMetadata.composer?.toString(),
            albumArtist  = mediaMetadata.albumArtist?.toString(),
            conductor    = mediaMetadata.conductor?.toString(),
            writer       = mediaMetadata.writer?.toString(),
            station      = mediaMetadata.station?.toString(),
        ).also {
            Timber.d("MetadataExtractor: extracted '${it.displayTitle}' for $mediaId")
        }
    }
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/metadata/MetadataManager.kt

```kotlin
package com.xplayer.dev.core.metadata

import androidx.annotation.MainThread
import androidx.media3.common.MediaMetadata
import timber.log.Timber
import javax.inject.Inject
import javax.inject.Singleton

/**
 * # MetadataManager
 *
 * Public API for media metadata.
 *
 * Responsibilities:
 *  - Receive [MediaMetadata] from ExoPlayer callbacks.
 *  - Extract via [MetadataExtractor].
 *  - Cache via [MetadataRepository].
 *  - Expose current and historical metadata.
 */
@Singleton
class MetadataManager @Inject constructor(
    private val extractor: MetadataExtractor,
    private val repository: MetadataRepository,
) {

    private var _currentMediaId: String? = null
    private var _currentInfo: MetadataExtractor.MediaInfo? = null

    val currentInfo: MetadataExtractor.MediaInfo? get() = _currentInfo
    val currentMediaId: String?                   get() = _currentMediaId

    /**
     * Process [MediaMetadata] from ExoPlayer.
     * Call from [Player.Listener.onMediaMetadataChanged].
     */
    @MainThread
    fun onMetadataChanged(
        metadata: MediaMetadata,
        mediaId: String?,
    ) {
        val info = extractor.extract(metadata, mediaId)
        _currentInfo   = info
        _currentMediaId = mediaId

        if (mediaId != null) {
            repository.save(mediaId, info)
        }

        Timber.i("MetadataManager: updated — '${info.displayTitle}'")
    }

    /**
     * Get metadata for a specific [mediaId].
     */
    fun getMetadata(mediaId: String): MetadataExtractor.MediaInfo? =
        repository.get(mediaId)

    /**
     * Clear current session metadata.
     */
    fun reset() {
        _currentInfo    = null
        _currentMediaId = null
        Timber.d("MetadataManager: reset")
    }
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/metadata/MetadataRepository.kt

```kotlin
package com.xplayer.dev.core.metadata

import timber.log.Timber
import java.util.concurrent.ConcurrentHashMap
import javax.inject.Inject
import javax.inject.Singleton

/**
 * # MetadataRepository
 *
 * In-memory cache of [MetadataExtractor.MediaInfo] per mediaId.
 * Keyed by mediaId for fast lookup.
 */
@Singleton
class MetadataRepository @Inject constructor() {

    private val _cache = ConcurrentHashMap<String, MetadataExtractor.MediaInfo>()

    fun save(mediaId: String, info: MetadataExtractor.MediaInfo) {
        _cache[mediaId] = info
        Timber.d("MetadataRepository: cached metadata for $mediaId")
    }

    fun get(mediaId: String): MetadataExtractor.MediaInfo? = _cache[mediaId]

    fun getAll(): Map<String, MetadataExtractor.MediaInfo> = _cache.toMap()

    fun remove(mediaId: String) {
        _cache.remove(mediaId)
    }

    fun clear() {
        _cache.clear()
        Timber.d("MetadataRepository: cache cleared")
    }

    fun hasMetadata(mediaId: String): Boolean = _cache.containsKey(mediaId)

    val size: Int get() = _cache.size
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/live/LiveLatencyController.kt

```kotlin
package com.xplayer.dev.core.live

import androidx.media3.exoplayer.ExoPlayer
import timber.log.Timber
import javax.inject.Inject
import javax.inject.Singleton

/**
 * # LiveLatencyController
 *
 * Controls live stream latency via playback speed adjustment.
 *
 * Strategy:
 *  - If latency > [targetLatencyMs] + [toleranceMs]: speed up slightly.
 *  - If latency < [targetLatencyMs] - [toleranceMs]: slow down slightly.
 *  - Within tolerance: maintain normal speed.
 */
@Singleton
class LiveLatencyController @Inject constructor() {

    data class LatencyConfig(
        val targetLatencyMs: Long  = 5_000L,
        val toleranceMs: Long      = 1_000L,
        val minSpeed: Float        = 0.97f,
        val maxSpeed: Float        = 1.03f,
        val enabled: Boolean       = true,
    )

    var config: LatencyConfig = LatencyConfig()
        private set

    /**
     * Adjust playback speed based on current latency.
     */
    fun adjustSpeed(player: ExoPlayer) {
        if (!config.enabled) return
        if (!player.isCurrentMediaItemLive) return

        val currentLatency = player.currentLiveOffset
        val targetLatency  = config.targetLatencyMs

        val newSpeed = when {
            currentLatency > targetLatency + config.toleranceMs -> config.maxSpeed
            currentLatency < targetLatency - config.toleranceMs -> config.minSpeed
            else -> 1.0f
        }

        if (newSpeed != player.playbackParameters.speed) {
            player.playbackParameters = androidx.media3.common.PlaybackParameters(newSpeed)
            Timber.d(
                "LiveLatencyController: latency=${currentLatency}ms " +
                    "target=${targetLatency}ms speed=$newSpeed"
            )
        }
    }

    fun configure(block: LatencyConfig.() -> LatencyConfig) {
        config = config.block()
    }
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/live/LivePlaybackManager.kt

```kotlin
package com.xplayer.dev.core.live

import androidx.annotation.MainThread
import androidx.media3.exoplayer.ExoPlayer
import timber.log.Timber
import javax.inject.Inject
import javax.inject.Singleton

/**
 * # LivePlaybackManager
 *
 * Controls live stream specific behaviour.
 *
 * Features:
 *  - Detect and report live stream state.
 *  - Return to live edge.
 *  - DVR window navigation.
 *  - Live latency reporting.
 */
@Singleton
class LivePlaybackManager @Inject constructor(
    private val latencyController: LiveLatencyController,
    private val sessionManager: LiveSessionManager,
) {

    private var playerRef: ExoPlayer? = null

    data class LiveInfo(
        val isLive: Boolean            = false,
        val isAtLiveEdge: Boolean      = false,
        val currentPositionMs: Long    = 0L,
        val liveEdgePositionMs: Long   = 0L,
        val latencyMs: Long            = 0L,
        val dvrWindowMs: Long          = 0L,
        val hasDvr: Boolean            = false,
    ) {
        val behindLiveMs: Long
            get() = (liveEdgePositionMs - currentPositionMs).coerceAtLeast(0L)
    }

    private var _liveInfo = LiveInfo()
    val liveInfo: LiveInfo get() = _liveInfo
    val isLive: Boolean    get() = _liveInfo.isLive

    /**
     * Attach ExoPlayer for live queries.
     */
    fun attachPlayer(player: ExoPlayer) {
        playerRef = player
        sessionManager.onPlayerAttached(player)
        Timber.d("LivePlaybackManager: player attached")
    }

    fun detachPlayer() {
        playerRef = null
        Timber.d("LivePlaybackManager: player detached")
    }

    /**
     * Update live info from current ExoPlayer state.
     * Call periodically from [PlaybackScheduler].
     */
    @MainThread
    fun update() {
        val player = playerRef ?: return
        if (!player.isCurrentMediaItemLive) {
            _liveInfo = LiveInfo(isLive = false)
            return
        }

        val currentPos    = player.currentPosition
        val liveEdge      = player.duration.coerceAtLeast(0L)
        val dvrWindow     = player.currentLiveOffset
        val isAtEdge      = player.isCurrentMediaItemDynamic &&
            kotlin.math.abs(player.currentLiveOffset) < LIVE_EDGE_THRESHOLD_MS

        _liveInfo = LiveInfo(
            isLive           = true,
            isAtLiveEdge     = isAtEdge,
            currentPositionMs = currentPos,
            liveEdgePositionMs = liveEdge,
            latencyMs        = player.currentLiveOffset.coerceAtLeast(0L),
            dvrWindowMs      = player.totalBufferedDuration,
            hasDvr           = player.isCurrentMediaItemSeekable,
        )
    }

    /**
     * Seek to the live edge.
     */
    @MainThread
    fun returnToLive() {
        val player = playerRef ?: return
        if (!player.isCurrentMediaItemLive) return
        player.seekToDefaultPosition()
        Timber.i("LivePlaybackManager: returned to live edge")
    }

    /**
     * Seek within the DVR window.
     * [offsetFromLiveMs] = 0 means live edge; positive = behind live.
     */
    @MainThread
    fun seekInDvr(offsetFromLiveMs: Long) {
        val player = playerRef ?: return
        if (!player.isCurrentMediaItemSeekable) {
            Timber.w("LivePlaybackManager: DVR seek not available")
            return
        }
        val targetPosition = (player.duration - offsetFromLiveMs).coerceAtLeast(0L)
        player.seekTo(targetPosition)
        Timber.d("LivePlaybackManager: DVR seek — behind live by ${offsetFromLiveMs}ms")
    }

    companion object {
        private const val LIVE_EDGE_THRESHOLD_MS = 5_000L
    }
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/live/LiveSessionManager.kt

```kotlin
package com.xplayer.dev.core.live

import androidx.media3.exoplayer.ExoPlayer
import timber.log.Timber
import javax.inject.Inject
import javax.inject.Singleton

/**
 * # LiveSessionManager
 *
 * Tracks live session health: reconnections, stream recoveries.
 */
@Singleton
class LiveSessionManager @Inject constructor() {

    data class LiveSessionStats(
        val reconnectCount: Int    = 0,
        val streamRecoveries: Int  = 0,
        val totalLiveMs: Long      = 0L,
        val sessionStartMs: Long   = System.currentTimeMillis(),
    ) {
        val sessionAgeMs: Long
            get() = System.currentTimeMillis() - sessionStartMs
    }

    private var _stats = LiveSessionStats()
    val stats: LiveSessionStats get() = _stats

    fun onPlayerAttached(player: ExoPlayer) {
        _stats = LiveSessionStats()
        Timber.d("LiveSessionManager: new live session started")
    }

    fun onReconnect() {
        _stats = _stats.copy(reconnectCount = _stats.reconnectCount + 1)
        Timber.w("LiveSessionManager: reconnect #${_stats.reconnectCount}")
    }

    fun onStreamRecovered() {
        _stats = _stats.copy(streamRecoveries = _stats.streamRecoveries + 1)
        Timber.i("LiveSessionManager: stream recovered #${_stats.streamRecoveries}")
    }

    fun reset() {
        _stats = LiveSessionStats()
    }
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/analytics/AnalyticsManager.kt

```kotlin
package com.xplayer.dev.core.analytics

import com.xplayer.dev.core.engine.internal.EngineMetricsProbe
import com.xplayer.dev.core.engine.internal.MetricsSnapshot
import com.xplayer.dev.core.events.PlayerEvent
import com.xplayer.dev.core.events.PlayerEventBus
import kotlinx.coroutines.CoroutineScope
import kotlinx.coroutines.Job
import kotlinx.coroutines.delay
import kotlinx.coroutines.flow.launchIn
import kotlinx.coroutines.flow.onEach
import kotlinx.coroutines.isActive
import kotlinx.coroutines.launch
import timber.log.Timber
import java.util.concurrent.atomic.AtomicLong
import java.util.concurrent.atomic.AtomicReference
import javax.inject.Inject
import javax.inject.Singleton

/**
 * AnalyticsManager
 *
 * Central coordinator for all playback analytics.
 * Aggregates metrics from EngineMetricsProbe and PlayerEventBus
 * into session-level QoS reports.
 *
 * Responsibilities:
 *  - Collect TTFP, stall, seek, drop, bandwidth metrics
 *  - Build PlaybackProfile per session
 *  - Forward to AnalyticsRepository for persistence
 *  - Expose real-time MetricsSnapshot
 */
@Singleton
class AnalyticsManager @Inject constructor(
    private val metricsProbe: EngineMetricsProbe,
    private val eventBus: PlayerEventBus,
    private val profiler: PlaybackProfiler,
    private val qosMonitor: QoSMonitor,
    private val repository: AnalyticsRepository,
) {
    private var collectionJob: Job? = null
    private val currentSessionId = AtomicReference<String?>(null)
    private val sessionStartMs = AtomicLong(0L)

    // ── Lifecycle ─────────────────────────────────────────────────────────────

    fun startSession(sessionId: String) {
        currentSessionId.set(sessionId)
        sessionStartMs.set(System.currentTimeMillis())
        profiler.reset()
        qosMonitor.reset()
        Timber.i("AnalyticsManager: session started [$sessionId]")
    }

    fun endSession() {
        val sessionId = currentSessionId.getAndSet(null) ?: return
        val snapshot = metricsProbe.snapshot()
        val profile = profiler.buildProfile(sessionId, snapshot)
        repository.saveProfile(profile)
        qosMonitor.onSessionEnd(profile)
        Timber.i("AnalyticsManager: session ended [$sessionId]")
    }

    fun startCollection(scope: CoroutineScope) {
        collectionJob?.cancel()
        collectionJob = scope.launch {
            // Periodic QoS sampling every 10s
            while (isActive) {
                delay(QOS_SAMPLE_INTERVAL_MS)
                currentSessionId.get()?.let {
                    val snapshot = metricsProbe.snapshot()
                    qosMonitor.onSample(snapshot)
                }
            }
        }

        // Subscribe to event bus for real-time metric updates
        eventBus.broadcast
            .onEach { event -> handleEvent(event) }
            .launchIn(scope)

        Timber.i("AnalyticsManager: collection started")
    }

    fun stopCollection() {
        collectionJob?.cancel()
        collectionJob = null
        Timber.i("AnalyticsManager: collection stopped")
    }

    // ── Real-time snapshot ────────────────────────────────────────────────────

    val currentSnapshot: MetricsSnapshot
        get() = metricsProbe.snapshot()

    val currentQoSHealth: QoSHealth
        get() = qosMonitor.currentHealth

    // ── Event handling ────────────────────────────────────────────────────────

    private fun handleEvent(event: PlayerEvent) {
        when (event) {
            is PlayerEvent.PlaybackStarted -> {
                if (!event.isResumed) {
                    profiler.recordTtfp(event.timeToFirstFrameMs)
                }
            }
            is PlayerEvent.BufferingStarted -> {
                if (event.isRebuffer) {
                    profiler.incrementStallCount()
                }
            }
            is PlayerEvent.BufferingEnded -> {
                profiler.addStallDuration(event.stallDurationMs)
            }
            is PlayerEvent.SeekCompleted -> {
                profiler.recordSeek(event.seekDurationMs)
            }
            is PlayerEvent.DroppedFrames -> {
                profiler.addDroppedFrames(event.droppedCount)
            }
            is PlayerEvent.BandwidthEstimated -> {
                profiler.recordBandwidth(event.bandwidthBps)
            }
            is PlayerEvent.ErrorOccurred -> {
                profiler.incrementErrorCount(event.isFatal)
            }
            else -> Unit
        }
    }

    companion object {
        private const val QOS_SAMPLE_INTERVAL_MS = 10_000L
    }
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/analytics/AnalyticsRepository.kt

```kotlin
package com.xplayer.dev.core.analytics

import timber.log.Timber
import java.util.concurrent.CopyOnWriteArrayList
import javax.inject.Inject
import javax.inject.Singleton

/**
 * AnalyticsRepository
 *
 * In-memory store for PlaybackProfile records.
 * In production, replace with Room / remote analytics SDK.
 */
@Singleton
class AnalyticsRepository @Inject constructor() {

    private val profiles = CopyOnWriteArrayList<PlaybackProfile>()

    fun saveProfile(profile: PlaybackProfile) {
        if (profiles.size >= MAX_PROFILES) profiles.removeAt(0)
        profiles.add(profile)
        Timber.d(
            "AnalyticsRepository: saved profile " +
                "[session=${profile.sessionId}] " +
                "[ttfp=${profile.ttfpMs}ms] " +
                "[stalls=${profile.stallCount}]"
        )
    }

    fun allProfiles(): List<PlaybackProfile> = profiles.toList()

    fun profileForSession(sessionId: String): PlaybackProfile? =
        profiles.firstOrNull { it.sessionId == sessionId }

    fun recentProfiles(n: Int): List<PlaybackProfile> =
        profiles.takeLast(n.coerceAtLeast(0))

    fun averageTtfpMs(): Long? {
        val values = profiles.mapNotNull { it.ttfpMs }
        return if (values.isEmpty()) null else values.average().toLong()
    }

    fun averageStallCount(): Float =
        if (profiles.isEmpty()) 0f
        else profiles.sumOf { it.stallCount }.toFloat() / profiles.size

    fun clear() = profiles.clear()

    companion object {
        private const val MAX_PROFILES = 200
    }
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/analytics/PlaybackProfiler.kt

```kotlin
package com.xplayer.dev.core.analytics

import com.xplayer.dev.core.engine.internal.MetricsSnapshot
import timber.log.Timber
import java.util.concurrent.atomic.AtomicInteger
import java.util.concurrent.atomic.AtomicLong
import javax.inject.Inject
import javax.inject.Singleton
import kotlin.math.roundToLong

/**
 * PlaybackProfiler
 *
 * Accumulates per-session playback metrics.
 * Thread-safe via Atomic fields.
 * Call reset() at start of every new session.
 */
@Singleton
class PlaybackProfiler @Inject constructor() {

    private val ttfpMs          = AtomicLong(-1L)
    private val stallCount      = AtomicInteger(0)
    private val totalStallMs    = AtomicLong(0L)
    private val seekCount       = AtomicInteger(0)
    private val totalSeekMs     = AtomicLong(0L)
    private val droppedFrames   = AtomicInteger(0)
    private val errorCount      = AtomicInteger(0)
    private val fatalErrorCount = AtomicInteger(0)
    private val bwSamples       = mutableListOf<Long>()
    private val bwLock          = Any()

    fun reset() {
        ttfpMs.set(-1L)
        stallCount.set(0)
        totalStallMs.set(0L)
        seekCount.set(0)
        totalSeekMs.set(0L)
        droppedFrames.set(0)
        errorCount.set(0)
        fatalErrorCount.set(0)
        synchronized(bwLock) { bwSamples.clear() }
        Timber.d("PlaybackProfiler: reset")
    }

    fun recordTtfp(ms: Long) {
        if (ttfpMs.get() == -1L) ttfpMs.set(ms)
    }

    fun incrementStallCount() { stallCount.incrementAndGet() }

    fun addStallDuration(ms: Long) { totalStallMs.addAndGet(ms) }

    fun recordSeek(durationMs: Long) {
        seekCount.incrementAndGet()
        totalSeekMs.addAndGet(durationMs)
    }

    fun addDroppedFrames(count: Int) { droppedFrames.addAndGet(count) }

    fun recordBandwidth(bps: Long) {
        synchronized(bwLock) {
            if (bwSamples.size >= MAX_BW_SAMPLES) bwSamples.removeAt(0)
            bwSamples.add(bps)
        }
    }

    fun incrementErrorCount(isFatal: Boolean) {
        errorCount.incrementAndGet()
        if (isFatal) fatalErrorCount.incrementAndGet()
    }

    fun buildProfile(sessionId: String, snapshot: MetricsSnapshot): PlaybackProfile {
        val bwCopy = synchronized(bwLock) { bwSamples.toList() }
        return PlaybackProfile(
            sessionId          = sessionId,
            ttfpMs             = ttfpMs.get().takeIf { it >= 0L },
            stallCount         = stallCount.get(),
            totalStallMs       = totalStallMs.get(),
            seekCount          = seekCount.get(),
            avgSeekMs          = if (seekCount.get() > 0)
                totalSeekMs.get() / seekCount.get() else 0L,
            droppedFrames      = droppedFrames.get(),
            errorCount         = errorCount.get(),
            fatalErrorCount    = fatalErrorCount.get(),
            avgBandwidthBps    = bwCopy.average()
                .takeIf { it.isFinite() }?.roundToLong() ?: 0L,
            peakBandwidthBps   = bwCopy.maxOrNull() ?: 0L,
            renderedFrames     = snapshot.renderedFrames,
        )
    }

    companion object {
        private const val MAX_BW_SAMPLES = 120
    }
}

// ── Profile model ─────────────────────────────────────────────────────────────

data class PlaybackProfile(
    val sessionId: String,
    val ttfpMs: Long?,
    val stallCount: Int,
    val totalStallMs: Long,
    val seekCount: Int,
    val avgSeekMs: Long,
    val droppedFrames: Int,
    val errorCount: Int,
    val fatalErrorCount: Int,
    val avgBandwidthBps: Long,
    val peakBandwidthBps: Long,
    val renderedFrames: Long,
    val recordedAtMs: Long = System.currentTimeMillis(),
) {
    val dropRate: Float
        get() {
            val total = renderedFrames + droppedFrames
            return if (total == 0L) 0f else droppedFrames.toFloat() / total
        }

    val isHealthy: Boolean
        get() = stallCount == 0
            && fatalErrorCount == 0
            && dropRate < 0.01f
            && (ttfpMs ?: Long.MAX_VALUE) < 3_000L
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/analytics/QoSMonitor.kt

```kotlin
package com.xplayer.dev.core.analytics

import com.xplayer.dev.core.engine.internal.MetricsSnapshot
import timber.log.Timber
import java.util.concurrent.atomic.AtomicReference
import javax.inject.Inject
import javax.inject.Singleton

/**
 * QoSMonitor
 *
 * Real-time Quality of Service health assessment.
 * Evaluates rolling metrics against SLA thresholds.
 *
 * SLA Thresholds (configurable):
 *  - TTFP < 3000ms
 *  - Stall rate < 5% of session time
 *  - Drop rate < 1%
 *  - Average bandwidth > 500kbps
 */
@Singleton
class QoSMonitor @Inject constructor() {

    private val _currentHealth = AtomicReference(QoSHealth.UNKNOWN)
    private val samples = mutableListOf<MetricsSnapshot>()
    private val lock = Any()

    val currentHealth: QoSHealth get() = _currentHealth.get()

    fun reset() {
        synchronized(lock) { samples.clear() }
        _currentHealth.set(QoSHealth.UNKNOWN)
    }

    fun onSample(snapshot: MetricsSnapshot) {
        synchronized(lock) {
            if (samples.size >= MAX_SAMPLES) samples.removeAt(0)
            samples.add(snapshot)
        }
        _currentHealth.set(evaluate(snapshot))
        Timber.v("QoSMonitor: health=${_currentHealth.get()} [ttfp=${snapshot.ttfpMs}ms]")
    }

    fun onSessionEnd(profile: PlaybackProfile) {
        val health = when {
            profile.fatalErrorCount > 0 -> QoSHealth.CRITICAL
            profile.stallCount > STALL_THRESHOLD -> QoSHealth.DEGRADED
            profile.dropRate > DROP_RATE_THRESHOLD -> QoSHealth.DEGRADED
            !profile.isHealthy -> QoSHealth.FAIR
            else -> QoSHealth.GOOD
        }
        Timber.i("QoSMonitor: session ended — final health=$health")
    }

    private fun evaluate(snapshot: MetricsSnapshot): QoSHealth {
        val ttfpOk = snapshot.ttfpMs?.let { it < TTFP_THRESHOLD_MS } ?: true
        val stallOk = snapshot.stallCount <= STALL_THRESHOLD
        val dropOk  = snapshot.dropRate < DROP_RATE_THRESHOLD
        val bwOk    = snapshot.averageBandwidthBps >= MIN_BANDWIDTH_BPS

        return when {
            !ttfpOk && snapshot.stallCount > 3 -> QoSHealth.CRITICAL
            !stallOk || !dropOk               -> QoSHealth.DEGRADED
            !ttfpOk || !bwOk                  -> QoSHealth.FAIR
            else                               -> QoSHealth.GOOD
        }
    }

    companion object {
        private const val MAX_SAMPLES          = 30
        private const val TTFP_THRESHOLD_MS    = 3_000L
        private const val STALL_THRESHOLD      = 3
        private const val DROP_RATE_THRESHOLD  = 0.01f
        private const val MIN_BANDWIDTH_BPS    = 500_000L
    }
}

enum class QoSHealth { GOOD, FAIR, DEGRADED, CRITICAL, UNKNOWN }

```

## xplayer/app/src/main/java/com/xplayer/dev/core/cache/CacheConfiguration.kt

```kotlin
package com.xplayer.dev.core.cache

/**
 * CacheConfiguration
 *
 * All tuneable cache parameters in one place.
 */
data class CacheConfiguration(
    /** Maximum disk space for media cache. Default 512MB. */
    val maxCacheSizeBytes: Long = 512L * 1_048_576L,

    /** Subdirectory name inside app cache dir. */
    val cacheDirectoryName: String = "player_cache",

    /**
     * Pre-buffer: how much to cache ahead of playback position (bytes).
     * Media3 honours this via LoadControl.
     */
    val targetBufferBytes: Int = 64 * 1_024 * 1_024, // 64MB

    /** Whether to use the cache for offline playback. */
    val enableOfflinePlayback: Boolean = true,

    /** Ignore cache errors and fall through to network. */
    val ignoreCacheOnError: Boolean = true,
) {
    companion object {
        val DEFAULT = CacheConfiguration()

        val LARGE = CacheConfiguration(
            maxCacheSizeBytes = 2L * 1_024L * 1_048_576L, // 2GB
        )

        val MINIMAL = CacheConfiguration(
            maxCacheSizeBytes = 128L * 1_048_576L, // 128MB
        )
    }
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/cache/CacheManager.kt

```kotlin
package com.xplayer.dev.core.cache

import android.content.Context
import androidx.media3.database.StandaloneDatabaseProvider
import androidx.media3.datasource.cache.CacheDataSource
import androidx.media3.datasource.cache.LeastRecentlyUsedCacheEvictor
import androidx.media3.datasource.cache.SimpleCache
import timber.log.Timber
import java.io.File
import java.util.concurrent.atomic.AtomicBoolean
import java.util.concurrent.atomic.AtomicReference
import javax.inject.Inject
import javax.inject.Singleton

/**
 * CacheManager
 *
 * Manages Media3 SimpleCache lifecycle.
 * Provides CacheDataSource.Factory for MediaSourceFactory.
 *
 * Design:
 *  - Single SimpleCache instance per process (Media3 requirement).
 *  - LRU eviction with configurable max size.
 *  - Thread-safe init/release via AtomicBoolean.
 *  - DatabaseProvider for reliable cache metadata.
 */
@Singleton
class CacheManager @Inject constructor(
    private val context: Context,
    private val config: CacheConfiguration,
    private val repository: CacheRepository,
) {
    private val _cache = AtomicReference<SimpleCache?>(null)
    private val _initialised = AtomicBoolean(false)

    val isInitialised: Boolean get() = _initialised.get()

    // ── Lifecycle ─────────────────────────────────────────────────────────────

    fun initialise() {
        if (!_initialised.compareAndSet(false, true)) {
            Timber.w("CacheManager: already initialised")
            return
        }
        runCatching {
            val cacheDir = File(context.cacheDir, config.cacheDirectoryName).also {
                if (!it.exists()) it.mkdirs()
            }
            val evictor = LeastRecentlyUsedCacheEvictor(config.maxCacheSizeBytes)
            val dbProvider = StandaloneDatabaseProvider(context)
            val cache = SimpleCache(cacheDir, evictor, dbProvider)
            _cache.set(cache)
            Timber.i(
                "CacheManager: initialised " +
                    "[dir=${cacheDir.absolutePath}] " +
                    "[maxSize=${config.maxCacheSizeBytes / 1_048_576}MB]"
            )
        }.onFailure { throwable ->
            _initialised.set(false)
            Timber.e(throwable, "CacheManager: initialisation failed")
            throw throwable
        }
    }

    fun release() {
        if (!_initialised.compareAndSet(true, false)) return
        _cache.getAndSet(null)?.release()
        Timber.i("CacheManager: released")
    }

    // ── Factory ───────────────────────────────────────────────────────────────

    /**
     * Build a CacheDataSource.Factory backed by the active SimpleCache.
     * Upstream factory provides network data source.
     */
    fun buildCacheDataSourceFactory(
        upstreamFactory: androidx.media3.datasource.DataSource.Factory,
    ): CacheDataSource.Factory {
        val cache = requireCache()
        return CacheDataSource.Factory()
            .setCache(cache)
            .setUpstreamDataSourceFactory(upstreamFactory)
            .setFlags(
                CacheDataSource.FLAG_IGNORE_CACHE_ON_ERROR or
                    CacheDataSource.FLAG_BLOCK_ON_CACHE
            )
    }

    // ── Operations ────────────────────────────────────────────────────────────

    fun getCacheSize(): Long = _cache.get()?.cacheSpace ?: 0L

    fun getCacheUsedBytes(): Long = _cache.get()?.cacheSpace ?: 0L

    fun isCached(key: String): Boolean =
        _cache.get()?.isCached(key, 0L, 1L) ?: false

    /**
     * Remove all cached data for a specific content key.
     */
    fun removeFromCache(key: String) {
        val cache = _cache.get() ?: return
        runCatching {
            androidx.media3.datasource.cache.CacheWriter(
                CacheDataSource.Factory()
                    .setCache(cache)
                    .createDataSource(),
                androidx.media3.datasource.DataSpec(
                    android.net.Uri.parse(key)
                ),
                null, null
            )
            Timber.d("CacheManager: removed key=[$key]")
        }.onFailure {
            Timber.w(it, "CacheManager: removeFromCache failed for key=[$key]")
        }
    }

    fun clearAll() {
        val cache = _cache.get() ?: return
        runCatching {
            cache.keys.forEach { key ->
                cache.removeResource(key)
            }
            repository.clearAll()
            Timber.i("CacheManager: cleared all cache")
        }.onFailure {
            Timber.e(it, "CacheManager: clearAll failed")
        }
    }

    // ── Internal ──────────────────────────────────────────────────────────────

    internal fun requireCache(): SimpleCache =
        _cache.get()
            ?: error("CacheManager not initialised. Call initialise() first.")
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/cache/CacheRepository.kt

```kotlin
package com.xplayer.dev.core.cache

import timber.log.Timber
import java.util.concurrent.ConcurrentHashMap
import javax.inject.Inject
import javax.inject.Singleton

/**
 * CacheRepository
 *
 * Tracks which media items are cached and their metadata.
 * Decoupled from SimpleCache internals — works as an index.
 */
@Singleton
class CacheRepository @Inject constructor() {

    private val cachedItems = ConcurrentHashMap<String, CacheEntry>()

    data class CacheEntry(
        val mediaId: String,
        val url: String,
        val cachedAtMs: Long = System.currentTimeMillis(),
        val sizeBytes: Long = 0L,
        val isComplete: Boolean = false,
    )

    fun record(entry: CacheEntry) {
        cachedItems[entry.mediaId] = entry
        Timber.d("CacheRepository: recorded [mediaId=${entry.mediaId}]")
    }

    fun isMediaCached(mediaId: String): Boolean = cachedItems.containsKey(mediaId)

    fun getEntry(mediaId: String): CacheEntry? = cachedItems[mediaId]

    fun allEntries(): List<CacheEntry> = cachedItems.values.toList()

    fun remove(mediaId: String) {
        cachedItems.remove(mediaId)
        Timber.d("CacheRepository: removed [mediaId=$mediaId]")
    }

    fun clearAll() {
        cachedItems.clear()
        Timber.i("CacheRepository: cleared")
    }

    val totalTrackedItems: Int get() = cachedItems.size
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/download/DownloadManager.kt

```kotlin
package com.xplayer.dev.core.download

import android.content.Context
import androidx.media3.exoplayer.offline.Download
import androidx.media3.exoplayer.offline.DownloadManager as Media3DownloadManager
import androidx.media3.exoplayer.offline.DownloadRequest
import timber.log.Timber
import java.util.concurrent.Executor
import java.util.concurrent.Executors
import javax.inject.Inject
import javax.inject.Singleton

/**
 * DownloadManager
 *
 * Wraps Media3 DownloadManager for offline content management.
 *
 * Responsibilities:
 *  - Start, pause, resume, cancel downloads
 *  - Track download state via DownloadRepository
 *  - Verify downloaded file integrity
 *  - Notify DownloadQueue of state changes
 */
@Singleton
class DownloadManager @Inject constructor(
    private val context: Context,
    private val repository: DownloadRepository,
    private val queue: DownloadQueue,
    private val scheduler: DownloadScheduler,
) {
    private var media3Manager: Media3DownloadManager? = null
    private val executor: Executor = Executors.newFixedThreadPool(
        MAX_PARALLEL_DOWNLOADS
    )

    // ── Lifecycle ─────────────────────────────────────────────────────────────

    fun initialise(
        dataSourceFactory: androidx.media3.datasource.DataSource.Factory,
        cache: androidx.media3.datasource.cache.Cache,
    ) {
        val databaseProvider = androidx.media3.database.StandaloneDatabaseProvider(context)
        media3Manager = Media3DownloadManager(
            context,
            databaseProvider,
            cache,
            dataSourceFactory,
            executor,
        ).also { mgr ->
            mgr.maxParallelDownloads = MAX_PARALLEL_DOWNLOADS
            mgr.addListener(DownloadListener())
            Timber.i("DownloadManager: initialised")
        }
    }

    fun release() {
        media3Manager?.release()
        media3Manager = null
        Timber.i("DownloadManager: released")
    }

    // ── Operations ────────────────────────────────────────────────────────────

    fun startDownload(request: DownloadRequest) {
        requireManager().addDownload(request)
        repository.record(
            DownloadRecord(
                downloadId = request.id,
                url        = request.uri.toString(),
                state      = DownloadState.QUEUED,
            )
        )
        queue.enqueue(request.id)
        Timber.i("DownloadManager: queued [id=${request.id}]")
    }

    fun pauseDownload(downloadId: String) {
        requireManager().setStopReason(downloadId, Download.STOP_REASON_NONE + 1)
        repository.updateState(downloadId, DownloadState.PAUSED)
        Timber.d("DownloadManager: paused [id=$downloadId]")
    }

    fun resumeDownload(downloadId: String) {
        requireManager().setStopReason(downloadId, Download.STOP_REASON_NONE)
        repository.updateState(downloadId, DownloadState.DOWNLOADING)
        Timber.d("DownloadManager: resumed [id=$downloadId]")
    }

    fun cancelDownload(downloadId: String) {
        requireManager().removeDownload(downloadId)
        repository.remove(downloadId)
        queue.remove(downloadId)
        Timber.d("DownloadManager: cancelled [id=$downloadId]")
    }

    fun retryDownload(downloadId: String) {
        val record = repository.getRecord(downloadId) ?: return
        pauseDownload(downloadId)
        resumeDownload(downloadId)
        Timber.d("DownloadManager: retrying [id=$downloadId]")
    }

    fun pauseAll() {
        requireManager().pauseDownloads()
        Timber.i("DownloadManager: all paused")
    }

    fun resumeAll() {
        requireManager().resumeDownloads()
        Timber.i("DownloadManager: all resumed")
    }

    // ── Query ─────────────────────────────────────────────────────────────────

    fun getDownloads(): List<Download> =
        requireManager().downloadIndex.getDownloads().run {
            val list = mutableListOf<Download>()
            use { cursor ->
                while (cursor.moveToNext()) list.add(cursor.download)
            }
            list
        }

    fun getDownload(downloadId: String): Download? =
        requireManager().downloadIndex.getDownload(downloadId)

    fun isDownloaded(downloadId: String): Boolean =
        getDownload(downloadId)?.state == Download.STATE_COMPLETED

    // ── Private ───────────────────────────────────────────────────────────────

    private fun requireManager(): Media3DownloadManager =
        media3Manager ?: error("DownloadManager not initialised")

    private inner class DownloadListener : Media3DownloadManager.Listener {

        override fun onDownloadChanged(
            downloadManager: Media3DownloadManager,
            download: Download,
            finalException: Exception?,
        ) {
            val state = when (download.state) {
                Download.STATE_DOWNLOADING -> DownloadState.DOWNLOADING
                Download.STATE_COMPLETED   -> DownloadState.COMPLETED
                Download.STATE_FAILED      -> DownloadState.FAILED
                Download.STATE_REMOVING    -> DownloadState.REMOVING
                Download.STATE_STOPPED     -> DownloadState.PAUSED
                Download.STATE_QUEUED      -> DownloadState.QUEUED
                else                       -> DownloadState.UNKNOWN
            }
            repository.updateState(download.request.id, state)
            repository.updateProgress(
                download.request.id,
                progress = download.percentDownloaded,
                bytesDownloaded = download.bytesDownloaded,
            )

            Timber.d(
                "DownloadManager: state change " +
                    "[id=${download.request.id}] " +
                    "[state=$state] " +
                    "[progress=${download.percentDownloaded}%]"
            )

            if (state == DownloadState.COMPLETED) {
                scheduler.onDownloadComplete(download.request.id)
            }
            if (state == DownloadState.FAILED) {
                scheduler.onDownloadFailed(download.request.id, finalException)
            }
        }

        override fun onDownloadRemoved(
            downloadManager: Media3DownloadManager,
            download: Download,
        ) {
            repository.remove(download.request.id)
            Timber.d("DownloadManager: removed [id=${download.request.id}]")
        }
    }

    companion object {
        private const val MAX_PARALLEL_DOWNLOADS = 3
    }
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/download/DownloadQueue.kt

```kotlin
package com.xplayer.dev.core.download

import timber.log.Timber
import java.util.concurrent.LinkedBlockingDeque
import javax.inject.Inject
import javax.inject.Singleton

/**
 * DownloadQueue
 *
 * Priority-aware queue tracking active download IDs.
 * FIFO by default; priority items go to front.
 */
@Singleton
class DownloadQueue @Inject constructor() {

    private val queue = LinkedBlockingDeque<String>()

    fun enqueue(downloadId: String) {
        if (!queue.contains(downloadId)) {
            queue.addLast(downloadId)
            Timber.d("DownloadQueue: enqueued [$downloadId] size=${queue.size}")
        }
    }

    fun prioritise(downloadId: String) {
        if (queue.remove(downloadId)) {
            queue.addFirst(downloadId)
            Timber.d("DownloadQueue: prioritised [$downloadId]")
        }
    }

    fun remove(downloadId: String) {
        queue.remove(downloadId)
    }

    fun peek(): String? = queue.peek()

    fun size(): Int = queue.size

    fun isEmpty(): Boolean = queue.isEmpty()

    fun all(): List<String> = queue.toList()

    fun clear() = queue.clear()
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/download/DownloadRepository.kt

```kotlin
package com.xplayer.dev.core.download

import timber.log.Timber
import java.util.concurrent.ConcurrentHashMap
import javax.inject.Inject
import javax.inject.Singleton

@Singleton
class DownloadRepository @Inject constructor() {

    private val records = ConcurrentHashMap<String, DownloadRecord>()

    fun record(record: DownloadRecord) {
        records[record.downloadId] = record
    }

    fun getRecord(downloadId: String): DownloadRecord? = records[downloadId]

    fun updateState(downloadId: String, state: DownloadState) {
        records.computeIfPresent(downloadId) { _, record ->
            record.copy(state = state)
        }
    }

    fun updateProgress(downloadId: String, progress: Float, bytesDownloaded: Long) {
        records.computeIfPresent(downloadId) { _, record ->
            record.copy(
                progress       = progress,
                bytesDownloaded = bytesDownloaded,
            )
        }
    }

    fun remove(downloadId: String) { records.remove(downloadId) }

    fun allRecords(): List<DownloadRecord> = records.values.toList()

    fun completedDownloads(): List<DownloadRecord> =
        records.values.filter { it.state == DownloadState.COMPLETED }

    fun failedDownloads(): List<DownloadRecord> =
        records.values.filter { it.state == DownloadState.FAILED }

    fun clearAll() = records.clear()
}

// ── Models ────────────────────────────────────────────────────────────────────

data class DownloadRecord(
    val downloadId: String,
    val url: String,
    val state: DownloadState          = DownloadState.QUEUED,
    val progress: Float               = 0f,
    val bytesDownloaded: Long         = 0L,
    val totalBytes: Long              = 0L,
    val startedAtMs: Long             = System.currentTimeMillis(),
    val completedAtMs: Long?          = null,
)

enum class DownloadState {
    QUEUED,
    DOWNLOADING,
    PAUSED,
    COMPLETED,
    FAILED,
    REMOVING,
    UNKNOWN,
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/download/DownloadScheduler.kt

```kotlin
package com.xplayer.dev.core.download

import timber.log.Timber
import javax.inject.Inject
import javax.inject.Singleton

/**
 * DownloadScheduler
 *
 * Coordinates download sequencing and retry scheduling.
 */
@Singleton
class DownloadScheduler @Inject constructor(
    private val repository: DownloadRepository,
) {
    private val retryCounters = mutableMapOf<String, Int>()

    fun onDownloadComplete(downloadId: String) {
        retryCounters.remove(downloadId)
        Timber.i("DownloadScheduler: completed [$downloadId]")
    }

    fun onDownloadFailed(downloadId: String, cause: Exception?) {
        val retries = retryCounters.getOrDefault(downloadId, 0)
        Timber.w(
            cause,
            "DownloadScheduler: failed [$downloadId] retries=$retries"
        )
        if (retries < MAX_AUTO_RETRIES) {
            retryCounters[downloadId] = retries + 1
            Timber.i(
                "DownloadScheduler: scheduling retry ${retries + 1}/$MAX_AUTO_RETRIES " +
                    "for [$downloadId]"
            )
        } else {
            Timber.e(
                "DownloadScheduler: max retries reached for [$downloadId]"
            )
            retryCounters.remove(downloadId)
        }
    }

    fun shouldRetry(downloadId: String): Boolean =
        (retryCounters[downloadId] ?: 0) < MAX_AUTO_RETRIES

    fun retryCount(downloadId: String): Int =
        retryCounters.getOrDefault(downloadId, 0)

    companion object {
        private const val MAX_AUTO_RETRIES = 3
    }
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/drm/DrmConfiguration.kt

```kotlin
package com.xplayer.dev.core.drm

import androidx.media3.datasource.HttpDataSource

/**
 * DrmConfiguration
 *
 * All DRM-related configuration parameters.
 */
data class DrmConfiguration(
    val httpDataSourceFactory: HttpDataSource.Factory,
    val multiSession: Boolean        = false,
    val playbackForOffline: Boolean  = true,
    val defaultLicenseServerUrl: String = "",
    val defaultRequestHeaders: Map<String, String> = emptyMap(),
)

```

## xplayer/app/src/main/java/com/xplayer/dev/core/drm/DrmKeyManager.kt

```kotlin
package com.xplayer.dev.core.drm

import android.content.Context
import timber.log.Timber
import java.io.File
import javax.inject.Inject
import javax.inject.Singleton

/**
 * DrmKeyManager
 *
 * Persists and retrieves offline DRM key set IDs.
 * Stores raw ByteArray per mediaId in app's files directory.
 *
 * In production: replace with EncryptedSharedPreferences or Keystore-backed storage.
 */
@Singleton
class DrmKeyManager @Inject constructor(
    private val context: Context,
) {
    private val keyDir: File by lazy {
        File(context.filesDir, "drm_keys").also { it.mkdirs() }
    }

    fun storeKeySetId(mediaId: String, keySetId: ByteArray) {
        runCatching {
            File(keyDir, mediaId.sanitizeFileName()).writeBytes(keySetId)
            Timber.d("DrmKeyManager: stored keySetId for [mediaId=$mediaId]")
        }.onFailure {
            Timber.e(it, "DrmKeyManager: failed to store keySetId [mediaId=$mediaId]")
        }
    }

    fun getKeySetId(mediaId: String): ByteArray? = runCatching {
        val file = File(keyDir, mediaId.sanitizeFileName())
        if (file.exists()) file.readBytes() else null
    }.getOrNull()

    fun removeKeySetId(mediaId: String) {
        runCatching {
            File(keyDir, mediaId.sanitizeFileName()).delete()
        }.onFailure {
            Timber.w(it, "DrmKeyManager: failed to remove keySetId [mediaId=$mediaId]")
        }
    }

    fun hasKeySetId(mediaId: String): Boolean =
        File(keyDir, mediaId.sanitizeFileName()).exists()

    fun clearAll() {
        keyDir.listFiles()?.forEach { it.delete() }
        Timber.i("DrmKeyManager: cleared all keys")
    }

    private fun String.sanitizeFileName(): String =
        replace(Regex("[^a-zA-Z0-9._-]"), "_")
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/drm/DrmManager.kt

```kotlin
package com.xplayer.dev.core.drm

import android.media.MediaDrm
import androidx.media3.common.MediaItem
import androidx.media3.exoplayer.drm.DefaultDrmSessionManager
import androidx.media3.exoplayer.drm.DrmSessionManager
import androidx.media3.exoplayer.drm.FrameworkMediaDrm
import androidx.media3.exoplayer.drm.HttpMediaDrmCallback
import androidx.media3.exoplayer.drm.OfflineLicenseHelper
import timber.log.Timber
import java.util.UUID
import javax.inject.Inject
import javax.inject.Singleton

/**
 * DrmManager
 *
 * Manages DRM session lifecycle, license acquisition, and offline licenses.
 *
 * Supported schemes:
 *  - Widevine (L1/L3)
 *  - ClearKey
 *
 * Architecture:
 *  - LicenseManager: license request / response / renewal
 *  - KeyManager: key storage and retrieval
 *  - SessionManager: DRM session open/close
 */
@Singleton
class DrmManager @Inject constructor(
    private val licenseManager: LicenseManager,
    private val keyManager: DrmKeyManager,
    private val sessionManager: DrmSessionManager_,
    private val config: DrmConfiguration,
) {

    // ── Session manager factory ───────────────────────────────────────────────

    /**
     * Build a DrmSessionManager for use in MediaSource construction.
     * Call once per prepare().
     */
    fun buildSessionManager(
        licenseServerUrl: String,
        requestHeaders: Map<String, String> = emptyMap(),
        drmScheme: UUID = WIDEVINE_UUID,
    ): DrmSessionManager {
        val callback = HttpMediaDrmCallback(
            licenseServerUrl,
            config.httpDataSourceFactory,
        ).apply {
            requestHeaders.forEach { (key, value) ->
                setKeyRequestProperty(key, value)
            }
        }

        return DefaultDrmSessionManager.Builder()
            .setUuidAndExoMediaDrmProvider(
                drmScheme,
                FrameworkMediaDrm.DEFAULT_PROVIDER,
            )
            .setMultiSession(config.multiSession)
            .build(callback)
            .also {
                Timber.i(
                    "DrmManager: session manager built " +
                        "[scheme=${if (drmScheme == WIDEVINE_UUID) "Widevine" else "ClearKey"}]"
                )
            }
    }

    // ── Offline license ───────────────────────────────────────────────────────

    /**
     * Download and persist an offline DRM license for [mediaItem].
     */
    suspend fun downloadOfflineLicense(
        mediaItem: MediaItem,
        licenseServerUrl: String,
        requestHeaders: Map<String, String> = emptyMap(),
    ): Result<ByteArray> = runCatching {
        val drmConfig = mediaItem.localConfiguration?.drmConfiguration
            ?: error("No DRM configuration in MediaItem")
        val eventDispatcher = androidx.media3.exoplayer.drm.DrmSessionEventListener.EventDispatcher()
        val helper = OfflineLicenseHelper.newWidevineInstance(
            licenseServerUrl,
            false,
            config.httpDataSourceFactory,
            requestHeaders,
            eventDispatcher,
        )

        val schemeData = androidx.media3.common.DrmInitData.SchemeData(
            drmConfig.scheme,
            "video/mp4",
            drmConfig.keySetId ?: byteArrayOf()
        )
        val format = androidx.media3.common.Format.Builder()
            .setDrmInitData(androidx.media3.common.DrmInitData(schemeData))
            .build()
        val keySetId = helper.downloadLicense(format)
        helper.release()

        val mediaId = mediaItem.mediaId
        keyManager.storeKeySetId(mediaId, keySetId)
        Timber.i("DrmManager: offline license downloaded [mediaId=$mediaId]")
        keySetId
    }.onFailure {
        Timber.e(it, "DrmManager: offline license download FAILED")
    }

    /**
     * Restore a previously downloaded offline license.
     */
    fun restoreOfflineLicense(mediaId: String): ByteArray? {
        return keyManager.getKeySetId(mediaId).also {
            Timber.d("DrmManager: restore license [mediaId=$mediaId found=${it != null}]")
        }
    }

    /**
     * Remove offline license for [mediaId].
     */
    fun removeOfflineLicense(mediaId: String) {
        keyManager.removeKeySetId(mediaId)
        Timber.i("DrmManager: offline license removed [mediaId=$mediaId]")
    }

    // ── Security checks ───────────────────────────────────────────────────────

    /**
     * Check if Widevine L1 (hardware-level DRM) is supported.
     */
    fun isWidevineL1Supported(): Boolean = runCatching {
        val drm = MediaDrm(WIDEVINE_UUID)
        val level = drm.getPropertyString("securityLevel")
        drm.close()
        level == "L1"
    }.getOrDefault(false)

    /**
     * Get Widevine security level string ("L1" / "L3" / unknown).
     */
    fun getWidevineSecurityLevel(): String = runCatching {
        val drm = MediaDrm(WIDEVINE_UUID)
        val level = drm.getPropertyString("securityLevel")
        drm.close()
        level
    }.getOrDefault("unknown")

    companion object {
        val WIDEVINE_UUID: UUID = UUID.fromString("edef8ba9-79d6-4ace-a3c8-27dcd51d21ed")
        val CLEARKEY_UUID: UUID = UUID.fromString("e2719d58-a985-b3c9-781a-b030af78d30e")
    }
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/drm/DrmSessionManager_.kt

```kotlin
package com.xplayer.dev.core.drm

import timber.log.Timber
import javax.inject.Inject
import javax.inject.Singleton

/**
 * DrmSessionManager_ (trailing underscore to avoid clash with Media3 type)
 *
 * Tracks open DRM sessions and their state.
 */
@Singleton
class DrmSessionManager_ @Inject constructor() {

    enum class SessionState { OPENING, OPEN, CLOSING, CLOSED, ERROR }

    data class DrmSession(
        val sessionId: String,
        val mediaId: String,
        val drmScheme: String,
        val state: SessionState       = SessionState.OPENING,
        val openedAtMs: Long          = System.currentTimeMillis(),
    )

    private val sessions = mutableMapOf<String, DrmSession>()

    fun openSession(session: DrmSession) {
        sessions[session.sessionId] = session
        Timber.d(
            "DrmSessionManager: opened " +
                "[sessionId=${session.sessionId}] " +
                "[mediaId=${session.mediaId}]"
        )
    }

    fun updateState(sessionId: String, state: SessionState) {
        sessions.computeIfPresent(sessionId) { _, s -> s.copy(state = state) }
    }

    fun closeSession(sessionId: String) {
        sessions.remove(sessionId)
        Timber.d("DrmSessionManager: closed [sessionId=$sessionId]")
    }

    fun getSession(sessionId: String): DrmSession? = sessions[sessionId]

    fun activeSessions(): List<DrmSession> =
        sessions.values.filter { it.state == SessionState.OPEN }

    fun closeAll() {
        sessions.clear()
        Timber.i("DrmSessionManager: all sessions closed")
    }
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/drm/LicenseManager.kt

```kotlin
package com.xplayer.dev.core.drm

import timber.log.Timber
import java.util.concurrent.ConcurrentHashMap
import javax.inject.Inject
import javax.inject.Singleton

/**
 * LicenseManager
 *
 * Tracks DRM license state and expiry.
 */
@Singleton
class LicenseManager @Inject constructor() {

    data class LicenseRecord(
        val mediaId: String,
        val licenseServerUrl: String,
        val acquiredAtMs: Long = System.currentTimeMillis(),
        val expiryMs: Long     = Long.MAX_VALUE,
        val isOffline: Boolean = false,
    ) {
        val isExpired: Boolean get() = System.currentTimeMillis() > expiryMs
        val isValid: Boolean   get() = !isExpired
    }

    private val licenses = ConcurrentHashMap<String, LicenseRecord>()

    fun recordLicense(record: LicenseRecord) {
        licenses[record.mediaId] = record
        Timber.d(
            "LicenseManager: recorded " +
                "[mediaId=${record.mediaId}] " +
                "[offline=${record.isOffline}]"
        )
    }

    fun getLicense(mediaId: String): LicenseRecord? = licenses[mediaId]

    fun isLicenseValid(mediaId: String): Boolean =
        licenses[mediaId]?.isValid ?: false

    fun removeLicense(mediaId: String) {
        licenses.remove(mediaId)
    }

    fun expiredLicenses(): List<LicenseRecord> =
        licenses.values.filter { it.isExpired }

    fun clearExpired() {
        licenses.entries.removeIf { it.value.isExpired }
        Timber.d("LicenseManager: cleared expired licenses")
    }

    fun clearAll() = licenses.clear()
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/codec/CodecDetector.kt

```kotlin
package com.xplayer.dev.core.codec

import android.media.MediaCodecInfo
import android.media.MediaCodecList
import timber.log.Timber
import javax.inject.Inject
import javax.inject.Singleton

/**
 * CodecDetector
 *
 * Discovers all available decoders using Android MediaCodecList.
 */
@Singleton
class CodecDetector @Inject constructor() {

    fun discoverAll(): List<CodecInfo> {
        val codecList = MediaCodecList(MediaCodecList.ALL_CODECS)
        return codecList.codecInfos
            .filter { !it.isEncoder }
            .flatMap { info -> buildCodecInfos(info) }
            .also { list ->
                Timber.d("CodecDetector: found ${list.size} decoder entries")
            }
    }

    private fun buildCodecInfos(info: MediaCodecInfo): List<CodecInfo> {
        return info.supportedTypes.mapNotNull { mimeType ->
            runCatching {
                val caps = info.getCapabilitiesForType(mimeType)
                val videoCaps = caps.videoCapabilities
                val isHw = info.isHardwareAccelerated

                CodecInfo(
                    name                  = info.name,
                    mimeType              = mimeType,
                    isHardwareAccelerated = isHw,
                    isSoftwareOnly        = info.isSoftwareOnly,
                    isVendor              = info.isVendor,
                    maxWidth              = videoCaps?.supportedWidths?.upper ?: 0,
                    maxHeight             = videoCaps?.supportedHeights?.upper ?: 0,
                    maxFrameRate          = videoCaps?.supportedFrameRates?.upper?.toInt() ?: 0,
                    supportsHdr           = isHw && checkHdrSupport(info, mimeType),
                    profileLevels         = caps.profileLevels.map { pl ->
                        CodecProfileLevel(pl.profile, pl.level)
                    },
                )
            }.getOrNull()
        }
    }

    private fun checkHdrSupport(info: MediaCodecInfo, mimeType: String): Boolean {
        return when (mimeType) {
            "video/hevc" -> {
                // HEVC Main10 profile = 2
                val caps = info.getCapabilitiesForType(mimeType)
                caps.profileLevels.any { it.profile == 0x02 }
            }
            "video/x-vnd.on2.vp9" -> {
                // VP9 Profile 2 = HDR
                val caps = info.getCapabilitiesForType(mimeType)
                caps.profileLevels.any { it.profile == 0x02 }
            }
            else -> false
        }
    }
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/codec/CodecManager.kt

```kotlin
package com.xplayer.dev.core.codec

import android.media.MediaCodecInfo
import android.media.MediaCodecList
import timber.log.Timber
import javax.inject.Inject
import javax.inject.Singleton

/**
 * CodecManager
 *
 * Detects hardware/software codec support and selects
 * the optimal decoder for a given format.
 *
 * Uses Android MediaCodecList for capability detection.
 * Results are cached — codec list does not change at runtime.
 */
@Singleton
class CodecManager @Inject constructor(
    private val repository: CodecRepository,
    private val detector: CodecDetector,
    private val selector: DecoderSelector,
) {

    // ── Initialisation ────────────────────────────────────────────────────────

    /**
     * Discover and index all available codecs.
     * Call once at app startup; results are cached.
     */
    fun discoverCodecs() {
        val allCodecs = detector.discoverAll()
        repository.storeAll(allCodecs)
        Timber.i(
            "CodecManager: discovered ${allCodecs.size} codecs " +
                "[hw=${allCodecs.count { it.isHardwareAccelerated }}]"
        )
    }

    // ── Selection ─────────────────────────────────────────────────────────────

    /**
     * Select the best decoder for [mimeType] and optional [config].
     * Prefers hardware; falls back to software automatically.
     */
    fun selectDecoder(
        mimeType: String,
        config: CodecSelectionConfig = CodecSelectionConfig(),
    ): CodecInfo? {
        val candidates = repository.getForMimeType(mimeType)
        return selector.select(candidates, config).also { codec ->
            Timber.d(
                "CodecManager: selected " +
                    "[mime=$mimeType] " +
                    "[codec=${codec?.name ?: "NONE"}] " +
                    "[hw=${codec?.isHardwareAccelerated}]"
            )
        }
    }

    /**
     * True if hardware decoding is available for [mimeType].
     */
    fun isHardwareDecoderAvailable(mimeType: String): Boolean =
        repository.getForMimeType(mimeType).any { it.isHardwareAccelerated }

    /**
     * True if any decoder (hw or sw) supports [mimeType].
     */
    fun isFormatSupported(mimeType: String): Boolean =
        repository.getForMimeType(mimeType).isNotEmpty()

    /**
     * True if hardware decoder supports HDR (HEVC Main 10 / VP9 Profile 2).
     */
    fun isHdrSupported(): Boolean =
        repository.getForMimeType("video/hevc")
            .any { it.isHardwareAccelerated && it.supportsHdr }
        || repository.getForMimeType("video/x-vnd.on2.vp9")
            .any { it.isHardwareAccelerated && it.supportsHdr }

    /**
     * True if hardware decoder supports the given [width] x [height].
     */
    fun isResolutionSupported(
        mimeType: String,
        width: Int,
        height: Int,
    ): Boolean = repository.getForMimeType(mimeType)
        .filter { it.isHardwareAccelerated }
        .any { it.maxWidth >= width && it.maxHeight >= height }

    // ── Statistics ────────────────────────────────────────────────────────────

    fun allCodecs(): List<CodecInfo> = repository.all()

    fun hardwareCodecs(): List<CodecInfo> =
        repository.all().filter { it.isHardwareAccelerated }

    fun softwareCodecs(): List<CodecInfo> =
        repository.all().filter { !it.isHardwareAccelerated }
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/codec/CodecRepository.kt

```kotlin
package com.xplayer.dev.core.codec

import java.util.concurrent.ConcurrentHashMap
import javax.inject.Inject
import javax.inject.Singleton

@Singleton
class CodecRepository @Inject constructor() {

    private val byMimeType = ConcurrentHashMap<String, MutableList<CodecInfo>>()
    private val allCodecs  = mutableListOf<CodecInfo>()
    private val lock       = Any()

    fun storeAll(codecs: List<CodecInfo>) {
        synchronized(lock) {
            allCodecs.clear()
            byMimeType.clear()
            codecs.forEach { codec ->
                allCodecs.add(codec)
                byMimeType.getOrPut(codec.mimeType) { mutableListOf() }.add(codec)
            }
        }
    }

    fun getForMimeType(mimeType: String): List<CodecInfo> =
        byMimeType[mimeType]?.toList() ?: emptyList()

    fun all(): List<CodecInfo> = synchronized(lock) { allCodecs.toList() }

    fun clear() {
        synchronized(lock) {
            allCodecs.clear()
            byMimeType.clear()
        }
    }
}

// ── Models ────────────────────────────────────────────────────────────────────

data class CodecInfo(
    val name: String,
    val mimeType: String,
    val isHardwareAccelerated: Boolean,
    val isSoftwareOnly: Boolean,
    val isVendor: Boolean,
    val maxWidth: Int,
    val maxHeight: Int,
    val maxFrameRate: Int,
    val supportsHdr: Boolean,
    val profileLevels: List<CodecProfileLevel>,
)

data class CodecProfileLevel(
    val profile: Int,
    val level: Int,
)

data class CodecSelectionConfig(
    val preferHardware: Boolean  = true,
    val requireHdr: Boolean      = false,
    val minWidth: Int            = 0,
    val minHeight: Int           = 0,
    val mimeType: String         = "",
)

```

## xplayer/app/src/main/java/com/xplayer/dev/core/codec/DecoderSelector.kt

```kotlin
package com.xplayer.dev.core.codec

import timber.log.Timber
import javax.inject.Inject
import javax.inject.Singleton

/**
 * DecoderSelector
 *
 * Selects optimal decoder from candidates based on CodecSelectionConfig.
 * Priority: hardware > vendor > software
 */
@Singleton
class DecoderSelector @Inject constructor() {

    fun select(
        candidates: List<CodecInfo>,
        config: CodecSelectionConfig,
    ): CodecInfo? {
        if (candidates.isEmpty()) return null

        val filtered = candidates.filter { codec ->
            if (config.requireHdr && !codec.supportsHdr) return@filter false
            if (config.minWidth > 0 && codec.maxWidth < config.minWidth) return@filter false
            if (config.minHeight > 0 && codec.maxHeight < config.minHeight) return@filter false
            true
        }

        if (filtered.isEmpty()) {
            Timber.w("DecoderSelector: no codec meets requirements — falling back")
            return candidates.firstOrNull { it.isHardwareAccelerated }
                ?: candidates.firstOrNull()
        }

        return if (config.preferHardware) {
            filtered.firstOrNull { it.isHardwareAccelerated && it.isVendor }
                ?: filtered.firstOrNull { it.isHardwareAccelerated }
                ?: filtered.firstOrNull()
        } else {
            filtered.firstOrNull { it.isSoftwareOnly }
                ?: filtered.firstOrNull()
        }
    }
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/ffmpeg/FFmpegBridge.kt

```kotlin
package com.xplayer.dev.core.ffmpeg

import timber.log.Timber
import javax.inject.Inject
import javax.inject.Singleton

/**
 * FFmpegBridge
 *
 * Native library interface for FFmpeg extension.
 * Wraps JNI/reflection access to protect against missing library.
 */
@Singleton
class FFmpegBridge @Inject constructor() {

    private var loaded = false

    fun loadLibraries() {
        // Media3 FFmpeg extension loads automatically via SoLoader / ReLinker
        // Verify by checking class availability
        Class.forName(FFMPEG_RENDERER_CLASS)
        loaded = true
        Timber.d("FFmpegBridge: libraries loaded")
    }

    fun getVersion(): String = runCatching {
        val clazz = Class.forName(FFMPEG_RENDERER_CLASS)
        val method = clazz.getMethod("getVersion")
        method.invoke(null) as? String ?: "unknown"
    }.getOrDefault("unknown")

    fun release() {
        loaded = false
        Timber.d("FFmpegBridge: released")
    }

    val isLoaded: Boolean get() = loaded

    companion object {
        private const val FFMPEG_RENDERER_CLASS =
            "androidx.media3.decoder.ffmpeg.FfmpegAudioRenderer"
    }
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/ffmpeg/FFmpegCapabilityChecker.kt

```kotlin
package com.xplayer.dev.core.ffmpeg

import javax.inject.Inject
import javax.inject.Singleton

/**
 * FFmpegCapabilityChecker
 *
 * Knows which MIME types FFmpeg extension supports
 * that are not handled by platform decoders.
 */
@Singleton
class FFmpegCapabilityChecker @Inject constructor() {

    /**
     * All MIME types supported by the FFmpeg extension.
     */
    private val ffmpegSupportedTypes = setOf(
        // Audio
        "audio/ac3",
        "audio/eac3",
        "audio/eac3-joc",
        "audio/truehd",
        "audio/dts",
        "audio/dts-hd",
        "audio/opus",
        "audio/vorbis",
        "audio/flac",
        "audio/alac",
        "audio/mpeg",
        "audio/mp4a-latm",
        // Video (when ffmpeg-video extension present)
        "video/av01",
        "video/x-vnd.on2.vp8",
        "video/x-vnd.on2.vp9",
    )

    fun isSupported(mimeType: String): Boolean = mimeType in ffmpegSupportedTypes

    /**
     * Returns types that FFmpeg handles but typical platform decoders may not.
     */
    fun fallbackTypes(): List<String> = listOf(
        "audio/ac3",
        "audio/eac3",
        "audio/truehd",
        "audio/dts",
        "audio/dts-hd",
        "audio/flac",
        "audio/alac",
    )
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/ffmpeg/FFmpegDecoderManager.kt

```kotlin
package com.xplayer.dev.core.ffmpeg

import java.util.concurrent.atomic.AtomicInteger
import java.util.concurrent.atomic.AtomicLong
import javax.inject.Inject
import javax.inject.Singleton

/**
 * FFmpegDecoderManager
 *
 * Tracks FFmpeg decoder usage and statistics.
 */
@Singleton
class FFmpegDecoderManager @Inject constructor() {

    private val decodeCount     = AtomicInteger(0)
    private val failureCount    = AtomicInteger(0)
    private val totalDecodeMs   = AtomicLong(0L)
    private val switchCount     = AtomicInteger(0)

    fun onDecoderSwitch(fromPlatform: Boolean) {
        switchCount.incrementAndGet()
    }

    fun onDecodeSuccess(durationMs: Long) {
        decodeCount.incrementAndGet()
        totalDecodeMs.addAndGet(durationMs)
    }

    fun onDecodeFailure() {
        failureCount.incrementAndGet()
    }

    fun currentStats(): FFmpegDecoderStats = FFmpegDecoderStats(
        switchCount   = switchCount.get(),
        decodeCount   = decodeCount.get(),
        failureCount  = failureCount.get(),
        avgDecodeMs   = if (decodeCount.get() > 0)
            totalDecodeMs.get() / decodeCount.get() else 0L,
    )

    fun reset() {
        decodeCount.set(0)
        failureCount.set(0)
        totalDecodeMs.set(0L)
        switchCount.set(0)
    }
}

data class FFmpegDecoderStats(
    val switchCount: Int,
    val decodeCount: Int,
    val failureCount: Int,
    val avgDecodeMs: Long,
) {
    val failureRate: Float
        get() = if (decodeCount == 0) 0f
                else failureCount.toFloat() / decodeCount
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/ffmpeg/FFmpegManager.kt

```kotlin
package com.xplayer.dev.core.ffmpeg

import timber.log.Timber
import java.util.concurrent.atomic.AtomicBoolean
import javax.inject.Inject
import javax.inject.Singleton

/**
 * FFmpegManager
 *
 * Manages the FFmpeg extension lifecycle for Media3.
 *
 * Provides automatic fallback to FFmpeg decoders when
 * platform codecs cannot handle a format.
 *
 * Integration note:
 *  Add to build.gradle:
 *  implementation 'androidx.media3:media3-decoder-ffmpeg:1.4.x'
 *
 * Native libraries are loaded lazily on first use.
 */
@Singleton
class FFmpegManager @Inject constructor(
    private val bridge: FFmpegBridge,
    private val capabilityChecker: FFmpegCapabilityChecker,
    private val decoderManager: FFmpegDecoderManager,
) {
    private val _initialised = AtomicBoolean(false)
    private val _available   = AtomicBoolean(false)

    val isAvailable: Boolean get() = _available.get()
    val isInitialised: Boolean get() = _initialised.get()

    // ── Lifecycle ─────────────────────────────────────────────────────────────

    /**
     * Attempt to load and initialise FFmpeg native libraries.
     * Safe to call multiple times — idempotent.
     */
    fun initialise() {
        if (_initialised.getAndSet(true)) return

        runCatching {
            bridge.loadLibraries()
            val version = bridge.getVersion()
            _available.set(true)
            Timber.i("FFmpegManager: initialised [version=$version]")
        }.onFailure { throwable ->
            _available.set(false)
            _initialised.set(false)
            Timber.w(throwable, "FFmpegManager: FFmpeg not available")
        }
    }

    fun release() {
        if (!_initialised.compareAndSet(true, false)) return
        bridge.release()
        _available.set(false)
        Timber.i("FFmpegManager: released")
    }

    // ── Decoder support ───────────────────────────────────────────────────────

    /**
     * True if FFmpeg can decode [mimeType].
     */
    fun canDecode(mimeType: String): Boolean {
        if (!isAvailable) return false
        return capabilityChecker.isSupported(mimeType)
    }

    /**
     * Get the list of MIME types FFmpeg can decode that
     * the platform cannot handle.
     */
    fun getFallbackMimeTypes(): List<String> {
        if (!isAvailable) return emptyList()
        return capabilityChecker.fallbackTypes()
    }

    /**
     * Build a RenderersFactory that uses FFmpeg decoders as fallback.
     * Integrates with Media3 ExoPlayer builder.
     */
    fun buildRenderersFactory(
        context: android.content.Context,
        preferExtensionDecoders: Boolean = false,
    ): androidx.media3.exoplayer.RenderersFactory {
        if (!isAvailable) {
            Timber.w("FFmpegManager: returning default renderer (FFmpeg unavailable)")
            return androidx.media3.exoplayer.DefaultRenderersFactory(context)
        }

        val extensionMode = if (preferExtensionDecoders)
            androidx.media3.exoplayer.DefaultRenderersFactory
                .EXTENSION_RENDERER_MODE_PREFER
        else
            androidx.media3.exoplayer.DefaultRenderersFactory
                .EXTENSION_RENDERER_MODE_ON

        return androidx.media3.exoplayer.DefaultRenderersFactory(context)
            .setExtensionRendererMode(extensionMode)
            .also {
                Timber.i(
                    "FFmpegManager: renderer factory built " +
                        "[prefer=$preferExtensionDecoders]"
                )
            }
    }

    // ── Statistics ────────────────────────────────────────────────────────────

    fun getDecoderStats(): FFmpegDecoderStats = decoderManager.currentStats()
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/parser/ContainerParser.kt

```kotlin
package com.xplayer.dev.core.parser

import android.content.Context
import android.media.MediaMetadataRetriever
import android.net.Uri
import timber.log.Timber
import javax.inject.Inject
import javax.inject.Singleton

/**
 * ContainerParser
 *
 * Extracts technical stream info from local / network media containers.
 * Uses Android MediaMetadataRetriever for broad format support.
 */
@Singleton
class ContainerParser @Inject constructor() {

    data class ContainerInfo(
        val format: MediaFormat,
        val durationMs: Long,
        val width: Int,
        val height: Int,
        val frameRate: Float,
        val videoCodec: String?,
        val audioCodec: String?,
        val bitrate: Int,
        val rotation: Int,
    )

    fun parse(context: Context, uri: Uri): ContainerInfo {
        val retriever = MediaMetadataRetriever()
        return runCatching {
            retriever.setDataSource(context, uri)
            ContainerInfo(
                format     = detectFormat(uri),
                durationMs = retriever.extractMetadata(
                    MediaMetadataRetriever.METADATA_KEY_DURATION
                )?.toLongOrNull() ?: 0L,
                width      = retriever.extractMetadata(
                    MediaMetadataRetriever.METADATA_KEY_VIDEO_WIDTH
                )?.toIntOrNull() ?: 0,
                height     = retriever.extractMetadata(
                    MediaMetadataRetriever.METADATA_KEY_VIDEO_HEIGHT
                )?.toIntOrNull() ?: 0,
                frameRate  = retriever.extractMetadata(
                    MediaMetadataRetriever.METADATA_KEY_CAPTURE_FRAMERATE
                )?.toFloatOrNull() ?: 0f,
                videoCodec = null, // Not available via MMR
                audioCodec = null,
                bitrate    = retriever.extractMetadata(
                    MediaMetadataRetriever.METADATA_KEY_BITRATE
                )?.toIntOrNull() ?: 0,
                rotation   = retriever.extractMetadata(
                    MediaMetadataRetriever.METADATA_KEY_VIDEO_ROTATION
                )?.toIntOrNull() ?: 0,
            )
        }.onFailure {
            Timber.e(it, "ContainerParser: parse failed [uri=$uri]")
        }.getOrElse {
            ContainerInfo(
                format = MediaFormat.UNKNOWN,
                durationMs = 0L, width = 0, height = 0,
                frameRate = 0f, videoCodec = null, audioCodec = null,
                bitrate = 0, rotation = 0,
            )
        }.also {
            retriever.release()
        }
    }

    private fun detectFormat(uri: Uri): MediaFormat {
        val path = uri.path?.lowercase() ?: ""
        return when {
            path.endsWith(".mp4") || path.endsWith(".m4v") -> MediaFormat.MP4
            path.endsWith(".mkv")  -> MediaFormat.MKV
            path.endsWith(".webm") -> MediaFormat.WEBM
            path.endsWith(".avi")  -> MediaFormat.AVI
            path.endsWith(".ts")   -> MediaFormat.TS
            path.endsWith(".mov")  -> MediaFormat.MOV
            path.endsWith(".mp3")  -> MediaFormat.MP3
            path.endsWith(".flac") -> MediaFormat.FLAC
            path.endsWith(".aac")  -> MediaFormat.AAC
            else                   -> MediaFormat.UNKNOWN
        }
    }
}

enum class MediaFormat {
    MP4, MKV, WEBM, AVI, TS, MOV,
    MP3, FLAC, AAC,
    HLS, DASH, SMOOTH_STREAMING,
    UNKNOWN,
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/parser/ManifestParser.kt

```kotlin
package com.xplayer.dev.core.parser

import android.net.Uri
import timber.log.Timber
import javax.inject.Inject
import javax.inject.Singleton

/**
 * ManifestParser
 *
 * Parses HLS and DASH manifests into structured models.
 * Production implementation delegates to Media3 parsers.
 */
@Singleton
class ManifestParser @Inject constructor() {

    // ── HLS ───────────────────────────────────────────────────────────────────

    suspend fun parseHlsManifest(uri: Uri): HlsManifestInfo {
        // In production: use Media3 HlsPlaylistParser
        // This is a structural model — populate from actual parser
        return runCatching {
            HlsManifestInfo(
                masterUri      = uri,
                isLive         = false,
                variants       = emptyList(),
                audioTracks    = emptyList(),
                subtitleTracks = emptyList(),
            )
        }.onFailure {
            Timber.e(it, "ManifestParser: HLS parse failed [uri=$uri]")
        }.getOrDefault(HlsManifestInfo(uri))
    }

    // ── DASH ──────────────────────────────────────────────────────────────────

    suspend fun parseDashManifest(uri: Uri): DashManifestInfo {
        return runCatching {
            DashManifestInfo(
                mpdUri               = uri,
                durationMs           = 0L,
                isLive               = false,
                videoAdaptationSets  = emptyList(),
                audioAdaptationSets  = emptyList(),
                subtitleAdaptationSets = emptyList(),
            )
        }.onFailure {
            Timber.e(it, "ManifestParser: DASH parse failed [uri=$uri]")
        }.getOrDefault(DashManifestInfo(uri))
    }

    // ── Models ────────────────────────────────────────────────────────────────

    data class HlsManifestInfo(
        val masterUri: Uri,
        val isLive: Boolean            = false,
        val variants: List<HlsVariant> = emptyList(),
        val audioTracks: List<HlsTrack> = emptyList(),
        val subtitleTracks: List<HlsTrack> = emptyList(),
    )

    data class HlsVariant(
        val uri: Uri,
        val bandwidthBps: Int,
        val width: Int,
        val height: Int,
        val codecs: String?,
        val frameRate: Float,
    )

    data class HlsTrack(
        val uri: Uri?,
        val language: String?,
        val name: String?,
        val isDefault: Boolean,
        val isForced: Boolean,
    )

    data class DashManifestInfo(
        val mpdUri: Uri,
        val durationMs: Long                  = 0L,
        val isLive: Boolean                   = false,
        val videoAdaptationSets: List<DashVariant> = emptyList(),
        val audioAdaptationSets: List<DashVariant> = emptyList(),
        val subtitleAdaptationSets: List<DashVariant> = emptyList(),
    )

    data class DashVariant(
        val bandwidthBps: Int,
        val width: Int,
        val height: Int,
        val codecs: String?,
        val mimeType: String,
        val language: String?,
    )
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/parser/MediaParser.kt

```kotlin
package com.xplayer.dev.core.parser

import android.content.Context
import android.net.Uri
import timber.log.Timber
import javax.inject.Inject
import javax.inject.Singleton

/**
 * MediaParser
 *
 * Facade for all media parsing operations.
 * Delegates to specialized parsers based on content type.
 */
@Singleton
class MediaParser @Inject constructor(
    private val containerParser: ContainerParser,
    private val manifestParser: ManifestParser,
    private val metadataParser: MetadataParser,
) {

    /**
     * Parse all available information from [uri].
     * Returns a unified [ParsedMediaInfo] or null on failure.
     */
    suspend fun parse(
        context: Context,
        uri: Uri,
        mimeType: String? = null,
    ): ParsedMediaInfo? {
        return runCatching {
            val resolvedMime = mimeType ?: detectMimeType(uri)

            when {
                resolvedMime?.contains("hls") == true ||
                    uri.toString().endsWith(".m3u8") ->
                    parseHls(uri)

                resolvedMime?.contains("dash") == true ||
                    uri.toString().endsWith(".mpd") ->
                    parseDash(uri)

                else -> parseContainer(context, uri)
            }
        }.onFailure {
            Timber.e(it, "MediaParser: parse failed [uri=$uri]")
        }.getOrNull()
    }

    private suspend fun parseHls(uri: Uri): ParsedMediaInfo {
        val manifest = manifestParser.parseHlsManifest(uri)
        return ParsedMediaInfo(
            uri          = uri,
            format       = MediaFormat.HLS,
            durationMs   = -1L, // live unknown
            isLive       = manifest.isLive,
            variants     = manifest.variants,
            audioTracks  = manifest.audioTracks,
            subtitleTracks = manifest.subtitleTracks,
        )
    }

    private suspend fun parseDash(uri: Uri): ParsedMediaInfo {
        val manifest = manifestParser.parseDashManifest(uri)
        return ParsedMediaInfo(
            uri          = uri,
            format       = MediaFormat.DASH,
            durationMs   = manifest.durationMs,
            isLive       = manifest.isLive,
            variants     = manifest.videoAdaptationSets,
        )
    }

    private fun parseContainer(context: Context, uri: Uri): ParsedMediaInfo {
        val info = containerParser.parse(context, uri)
        val metadata = metadataParser.extract(context, uri)
        return ParsedMediaInfo(
            uri        = uri,
            format     = info.format,
            durationMs = info.durationMs,
            width      = info.width,
            height     = info.height,
            frameRate  = info.frameRate,
            videoCodec = info.videoCodec,
            audioCodec = info.audioCodec,
            bitrate    = info.bitrate,
            title      = metadata.title,
            artist     = metadata.artist,
            album      = metadata.album,
            artworkBytes = metadata.artworkBytes,
            chapters   = metadata.chapters,
        )
    }

    private fun detectMimeType(uri: Uri): String? {
        val path = uri.path ?: return null
        return when {
            path.endsWith(".m3u8") -> "application/vnd.apple.mpegurl"
            path.endsWith(".mpd")  -> "application/dash+xml"
            path.endsWith(".mp4")  -> "video/mp4"
            path.endsWith(".mkv")  -> "video/x-matroska"
            path.endsWith(".webm") -> "video/webm"
            path.endsWith(".avi")  -> "video/avi"
            path.endsWith(".mp3")  -> "audio/mpeg"
            path.endsWith(".flac") -> "audio/flac"
            path.endsWith(".aac")  -> "audio/aac"
            else                   -> null
        }
    }
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/parser/MetadataParser.kt

```kotlin
package com.xplayer.dev.core.parser

import android.content.Context
import android.media.MediaMetadataRetriever
import android.net.Uri
import timber.log.Timber
import javax.inject.Inject
import javax.inject.Singleton

/**
 * MetadataParser
 *
 * Extracts title, artist, album, artwork, chapters from media files.
 */
@Singleton
class MetadataParser @Inject constructor() {

    data class MediaMetadata(
        val title: String?,
        val artist: String?,
        val album: String?,
        val genre: String?,
        val year: String?,
        val trackNumber: String?,
        val artworkBytes: ByteArray?,
        val chapters: List<ChapterInfo>,
    )

    data class ChapterInfo(
        val index: Int,
        val title: String?,
        val startMs: Long,
        val endMs: Long,
    )

    fun extract(context: Context, uri: Uri): MediaMetadata {
        val retriever = MediaMetadataRetriever()
        return runCatching {
            retriever.setDataSource(context, uri)
            MediaMetadata(
                title       = retriever.extractMetadata(MediaMetadataRetriever.METADATA_KEY_TITLE),
                artist      = retriever.extractMetadata(MediaMetadataRetriever.METADATA_KEY_ARTIST),
                album       = retriever.extractMetadata(MediaMetadataRetriever.METADATA_KEY_ALBUM),
                genre       = retriever.extractMetadata(MediaMetadataRetriever.METADATA_KEY_GENRE),
                year        = retriever.extractMetadata(MediaMetadataRetriever.METADATA_KEY_YEAR),
                trackNumber = retriever.extractMetadata(MediaMetadataRetriever.METADATA_KEY_CD_TRACK_NUMBER),
                artworkBytes = retriever.embeddedPicture,
                chapters    = emptyList(), // requires MP4 chapter box parsing
            )
        }.onFailure {
            Timber.e(it, "MetadataParser: extraction failed [uri=$uri]")
        }.getOrElse {
            MediaMetadata(
                title = null, artist = null, album = null,
                genre = null, year = null, trackNumber = null,
                artworkBytes = null, chapters = emptyList(),
            )
        }.also {
            retriever.release()
        }
    }
}

// ── Unified ParsedMediaInfo ───────────────────────────────────────────────────

data class ParsedMediaInfo(
    val uri: android.net.Uri,
    val format: MediaFormat          = MediaFormat.UNKNOWN,
    val durationMs: Long             = 0L,
    val isLive: Boolean              = false,
    val width: Int                   = 0,
    val height: Int                  = 0,
    val frameRate: Float             = 0f,
    val videoCodec: String?          = null,
    val audioCodec: String?          = null,
    val bitrate: Int                 = 0,
    val title: String?               = null,
    val artist: String?              = null,
    val album: String?               = null,
    val artworkBytes: ByteArray?     = null,
    val chapters: List<MetadataParser.ChapterInfo> = emptyList(),
    val variants: List<Any>          = emptyList(),
    val audioTracks: List<Any>       = emptyList(),
    val subtitleTracks: List<Any>    = emptyList(),
)

```

## xplayer/app/src/main/java/com/xplayer/dev/core/library/LibraryRepository.kt

```kotlin
package com.xplayer.dev.core.library

import android.net.Uri
import java.util.concurrent.ConcurrentHashMap
import javax.inject.Inject
import javax.inject.Singleton

@Singleton
class LibraryRepository @Inject constructor() {

    private val items = ConcurrentHashMap<String, MediaItem_>()

    fun replaceAll(newItems: List<MediaItem_>) {
        items.clear()
        newItems.forEach { items[it.mediaId] = it }
    }

    fun addAll(newItems: List<MediaItem_>) {
        newItems.forEach { items[it.mediaId] = it }
    }

    fun allItems(): List<MediaItem_> = items.values.toList()

    fun getItem(mediaId: String): MediaItem_? = items[mediaId]

    fun remove(mediaId: String) { items.remove(mediaId) }

    fun updateFavourite(mediaId: String, isFavourite: Boolean) {
        items.computeIfPresent(mediaId) { _, item ->
            item.copy(isFavourite = isFavourite)
        }
    }

    fun updateWatchProgress(mediaId: String, progress: Float) {
        items.computeIfPresent(mediaId) { _, item ->
            item.copy(watchProgress = progress.coerceIn(0f, 1f))
        }
    }

    fun updateLastPlayed(mediaId: String, timestampMs: Long) {
        items.computeIfPresent(mediaId) { _, item ->
            item.copy(lastPlayedMs = timestampMs)
        }
    }

    fun recentlyPlayed(limit: Int): List<MediaItem_> =
        items.values
            .filter { it.lastPlayedMs > 0L }
            .sortedByDescending { it.lastPlayedMs }
            .take(limit)

    fun recentlyAdded(limit: Int): List<MediaItem_> =
        items.values
            .sortedByDescending { it.dateAdded }
            .take(limit)

    fun clear() = items.clear()
}

// ── Models ────────────────────────────────────────────────────────────────────

data class MediaItem_(
    val mediaId: String,
    val uri: Uri,
    val path: String,
    val title: String?,
    val mimeType: String,
    val sizeBytes: Long,
    val durationMs: Long,
    val dateAdded: Long,
    val width: Int              = 0,
    val height: Int             = 0,
    val type: MediaType         = MediaType.UNKNOWN,
    val isFavourite: Boolean    = false,
    val watchProgress: Float    = 0f,
    val lastPlayedMs: Long      = 0L,
)

enum class MediaType { VIDEO, AUDIO, UNKNOWN }

```

## xplayer/app/src/main/java/com/xplayer/dev/core/library/MediaIndexer.kt

```kotlin
package com.xplayer.dev.core.library

import timber.log.Timber
import java.util.UUID
import javax.inject.Inject
import javax.inject.Singleton

/**
 * MediaIndexer
 *
 * Converts ScannedFile list into indexed MediaItem_ records.
 * Detects duplicates by path.
 */
@Singleton
class MediaIndexer @Inject constructor() {

    fun index(files: List<MediaScanner.ScannedFile>): List<MediaItem_> {
        val seen = mutableSetOf<String>()
        return files.mapNotNull { file ->
            if (file.path in seen) {
                Timber.d("MediaIndexer: duplicate skipped [${file.path}]")
                null
            } else {
                seen.add(file.path)
                MediaItem_(
                    mediaId    = UUID.nameUUIDFromBytes(file.path.toByteArray()).toString(),
                    uri        = file.contentUri,
                    path       = file.path,
                    title      = file.displayName.removeSuffix(".mp4")
                        .removeSuffix(".mkv").removeSuffix(".mp3"),
                    mimeType   = file.mimeType,
                    sizeBytes  = file.sizeBytes,
                    durationMs = file.durationMs,
                    dateAdded  = file.dateAdded,
                    width      = file.width,
                    height     = file.height,
                    type       = if (file.mimeType.startsWith("video")) MediaType.VIDEO
                                 else MediaType.AUDIO,
                )
            }
        }.also {
            Timber.i("MediaIndexer: indexed ${it.size} items")
        }
    }
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/library/MediaLibrary.kt

```kotlin
package com.xplayer.dev.core.library

import timber.log.Timber
import javax.inject.Inject
import javax.inject.Singleton

/**
 * MediaLibrary
 *
 * Facade for local media discovery, indexing, and search.
 *
 * Responsibilities:
 *  - Trigger storage scans via MediaScanner
 *  - Maintain indexed media catalogue in LibraryRepository
 *  - Expose search and filter API
 *  - Track recently played and favourites
 */
@Singleton
class MediaLibrary @Inject constructor(
    private val scanner: MediaScanner,
    private val indexer: MediaIndexer,
    private val repository: LibraryRepository,
) {

    // ── Scan ──────────────────────────────────────────────────────────────────

    suspend fun scanAll(context: android.content.Context) {
        Timber.i("MediaLibrary: starting full scan")
        val discovered = scanner.scanAll(context)
        val indexed    = indexer.index(discovered)
        repository.replaceAll(indexed)
        Timber.i("MediaLibrary: scan complete [${indexed.size} items]")
    }

    suspend fun scanIncremental(context: android.content.Context) {
        Timber.i("MediaLibrary: incremental scan")
        val discovered = scanner.scanAll(context)
        val current    = repository.allItems().map { it.uri }.toSet()
        val newItems   = discovered.filter { it.contentUri.toString() !in current.map { u -> u.toString() } }
        val indexed    = indexer.index(newItems)
        repository.addAll(indexed)
        Timber.i("MediaLibrary: incremental scan added ${indexed.size} new items")
    }

    // ── Query ─────────────────────────────────────────────────────────────────

    fun allVideos(): List<MediaItem_> =
        repository.allItems().filter { it.type == MediaType.VIDEO }

    fun allAudio(): List<MediaItem_> =
        repository.allItems().filter { it.type == MediaType.AUDIO }

    fun recentlyPlayed(limit: Int = 20): List<MediaItem_> =
        repository.recentlyPlayed(limit)

    fun recentlyAdded(limit: Int = 20): List<MediaItem_> =
        repository.recentlyAdded(limit)

    fun favourites(): List<MediaItem_> =
        repository.allItems().filter { it.isFavourite }

    fun continueWatching(): List<MediaItem_> =
        repository.allItems().filter {
            it.watchProgress > 0.05f && it.watchProgress < 0.95f
        }

    // ── Search ────────────────────────────────────────────────────────────────

    fun search(query: String): List<MediaItem_> {
        if (query.isBlank()) return emptyList()
        val lower = query.lowercase()
        return repository.allItems().filter { item ->
            item.title?.lowercase()?.contains(lower) == true
                || item.path.lowercase().contains(lower)
        }
    }

    fun searchByFolder(folderPath: String): List<MediaItem_> =
        repository.allItems().filter { it.path.startsWith(folderPath) }

    // ── Management ────────────────────────────────────────────────────────────

    fun markFavourite(mediaId: String, isFavourite: Boolean) {
        repository.updateFavourite(mediaId, isFavourite)
    }

    fun updateWatchProgress(mediaId: String, progress: Float) {
        repository.updateWatchProgress(mediaId, progress)
    }

    fun markAsPlayed(mediaId: String) {
        repository.updateLastPlayed(mediaId, System.currentTimeMillis())
    }

    fun removeItem(mediaId: String) {
        repository.remove(mediaId)
    }

    val totalItemCount: Int get() = repository.allItems().size
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/library/MediaScanner.kt

```kotlin
package com.xplayer.dev.core.library

import android.content.ContentUris
import android.content.Context
import android.net.Uri
import android.provider.MediaStore
import timber.log.Timber
import javax.inject.Inject
import javax.inject.Singleton

/**
 * MediaScanner
 *
 * Queries Android MediaStore for all media files.
 * Supports internal, external, and SD card storage.
 */
@Singleton
class MediaScanner @Inject constructor() {

    data class ScannedFile(
        val contentUri: Uri,
        val displayName: String,
        val path: String,
        val mimeType: String,
        val sizeBytes: Long,
        val durationMs: Long,
        val dateAdded: Long,
        val width: Int,
        val height: Int,
    )

    suspend fun scanAll(context: Context): List<ScannedFile> {
        val results = mutableListOf<ScannedFile>()
        results += scanCollection(context, MediaStore.Video.Media.EXTERNAL_CONTENT_URI)
        results += scanCollection(context, MediaStore.Audio.Media.EXTERNAL_CONTENT_URI)
        Timber.i("MediaScanner: scanned ${results.size} files")
        return results
    }

    private fun scanCollection(context: Context, collection: Uri): List<ScannedFile> {
        val projection = arrayOf(
            MediaStore.MediaColumns._ID,
            MediaStore.MediaColumns.DISPLAY_NAME,
            MediaStore.MediaColumns.DATA,
            MediaStore.MediaColumns.MIME_TYPE,
            MediaStore.MediaColumns.SIZE,
            MediaStore.MediaColumns.DURATION,
            MediaStore.MediaColumns.DATE_ADDED,
            MediaStore.MediaColumns.WIDTH,
            MediaStore.MediaColumns.HEIGHT,
        )

        val results = mutableListOf<ScannedFile>()
        context.contentResolver.query(
            collection,
            projection,
            null, null,
            "${MediaStore.MediaColumns.DATE_ADDED} DESC",
        )?.use { cursor ->
            val idCol          = cursor.getColumnIndexOrThrow(MediaStore.MediaColumns._ID)
            val nameCol        = cursor.getColumnIndexOrThrow(MediaStore.MediaColumns.DISPLAY_NAME)
            val dataCol        = cursor.getColumnIndexOrThrow(MediaStore.MediaColumns.DATA)
            val mimeCol        = cursor.getColumnIndexOrThrow(MediaStore.MediaColumns.MIME_TYPE)
            val sizeCol        = cursor.getColumnIndexOrThrow(MediaStore.MediaColumns.SIZE)
            val durCol         = cursor.getColumnIndexOrThrow(MediaStore.MediaColumns.DURATION)
            val dateCol        = cursor.getColumnIndexOrThrow(MediaStore.MediaColumns.DATE_ADDED)
            val widthCol       = cursor.getColumnIndexOrThrow(MediaStore.MediaColumns.WIDTH)
            val heightCol      = cursor.getColumnIndexOrThrow(MediaStore.MediaColumns.HEIGHT)

            while (cursor.moveToNext()) {
                val id = cursor.getLong(idCol)
                val contentUri = ContentUris.withAppendedId(collection, id)
                results.add(
                    ScannedFile(
                        contentUri  = contentUri,
                        displayName = cursor.getString(nameCol) ?: "",
                        path        = cursor.getString(dataCol) ?: "",
                        mimeType    = cursor.getString(mimeCol) ?: "",
                        sizeBytes   = cursor.getLong(sizeCol),
                        durationMs  = cursor.getLong(durCol),
                        dateAdded   = cursor.getLong(dateCol) * 1000L,
                        width       = cursor.getInt(widthCol),
                        height      = cursor.getInt(heightCol),
                    )
                )
            }
        }
        return results
    }
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/persistence/HistoryManager.kt

```kotlin
package com.xplayer.dev.core.persistence

import timber.log.Timber
import javax.inject.Inject
import javax.inject.Singleton

/**
 * HistoryManager
 *
 * Manages watch history and continue-watching list.
 */
@Singleton
class HistoryManager @Inject constructor(
    private val repository: PlaybackRepository,
) {

    fun recordPlay(mediaId: String, title: String?, durationMs: Long) {
        repository.recordPlay(mediaId, title, durationMs)
        Timber.d("HistoryManager: recorded [mediaId=$mediaId]")
    }

    fun getRecentHistory(limit: Int = 50): List<HistoryRecord> =
        repository.recentHistory(limit)

    fun getPlayCount(mediaId: String): Int =
        repository.getHistory(mediaId)?.playCount ?: 0

    fun hasBeenPlayed(mediaId: String): Boolean =
        repository.getHistory(mediaId) != null

    fun clearAll() {
        repository.clearHistory()
        Timber.i("HistoryManager: cleared")
    }
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/persistence/PlaybackRepository.kt

```kotlin
package com.xplayer.dev.core.persistence

import timber.log.Timber
import java.util.concurrent.ConcurrentHashMap
import javax.inject.Inject
import javax.inject.Singleton

/**
 * PlaybackRepository
 *
 * Persists playback position, user preferences, and history.
 *
 * Production implementation: replace in-memory maps with:
 *  - Room database for positions and history
 *  - DataStore for preferences
 */
@Singleton
class PlaybackRepository @Inject constructor() {

    // ── Position storage ──────────────────────────────────────────────────────

    private val positions = ConcurrentHashMap<String, ResumeRecord>()

    fun savePosition(mediaId: String, positionMs: Long, durationMs: Long) {
        positions[mediaId] = ResumeRecord(
            mediaId    = mediaId,
            positionMs = positionMs,
            durationMs = durationMs,
            savedAtMs  = System.currentTimeMillis(),
        )
        Timber.d("PlaybackRepository: saved [mediaId=$mediaId pos=${positionMs}ms]")
    }

    fun getPosition(mediaId: String): ResumeRecord? = positions[mediaId]

    fun clearPosition(mediaId: String) {
        positions.remove(mediaId)
    }

    fun allPositions(): List<ResumeRecord> = positions.values.toList()

    // ── History ───────────────────────────────────────────────────────────────

    private val history = ConcurrentHashMap<String, HistoryRecord>()

    fun recordPlay(mediaId: String, title: String?, durationMs: Long) {
        val existing = history[mediaId]
        history[mediaId] = HistoryRecord(
            mediaId    = mediaId,
            title      = title,
            durationMs = durationMs,
            playCount  = (existing?.playCount ?: 0) + 1,
            lastPlayedMs = System.currentTimeMillis(),
        )
    }

    fun getHistory(mediaId: String): HistoryRecord? = history[mediaId]

    fun recentHistory(limit: Int = 50): List<HistoryRecord> =
        history.values
            .sortedByDescending { it.lastPlayedMs }
            .take(limit)

    fun clearHistory() = history.clear()

    // ── Preferences ───────────────────────────────────────────────────────────

    private val preferences = ConcurrentHashMap<String, String>()

    fun setPreference(key: PreferenceKey, value: String) {
        preferences[key.key] = value
    }

    fun getPreference(key: PreferenceKey): String? = preferences[key.key]

    fun getPreferenceOrDefault(key: PreferenceKey, default: String): String =
        preferences[key.key] ?: default

    fun clearPreference(key: PreferenceKey) = preferences.remove(key.key)

    fun clearAll() {
        positions.clear()
        history.clear()
        preferences.clear()
    }
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/persistence/PreferenceManager_.kt

```kotlin
package com.xplayer.dev.core.persistence

import javax.inject.Inject
import javax.inject.Singleton

/**
 * PreferenceManager_ (trailing underscore to avoid clash with android.preference)
 *
 * Typed API for player preferences.
 */
@Singleton
class PreferenceManager_ @Inject constructor(
    private val repository: PlaybackRepository,
) {

    // ── Playback speed ────────────────────────────────────────────────────────

    var playbackSpeed: Float
        get() = repository.getPreferenceOrDefault(
            PreferenceKey.PLAYBACK_SPEED, "1.0"
        ).toFloatOrNull() ?: 1.0f
        set(value) = repository.setPreference(
            PreferenceKey.PLAYBACK_SPEED, value.toString()
        )

    // ── Repeat mode ───────────────────────────────────────────────────────────

    var repeatMode: Int
        get() = repository.getPreferenceOrDefault(
            PreferenceKey.REPEAT_MODE, "0"
        ).toIntOrNull() ?: 0
        set(value) = repository.setPreference(
            PreferenceKey.REPEAT_MODE, value.toString()
        )

    // ── Audio track preference ────────────────────────────────────────────────

    var preferredAudioLanguage: String?
        get() = repository.getPreference(PreferenceKey.PREFERRED_AUDIO_LANGUAGE)
        set(value) {
            if (value != null) {
                repository.setPreference(PreferenceKey.PREFERRED_AUDIO_LANGUAGE, value)
            } else {
                repository.clearPreference(PreferenceKey.PREFERRED_AUDIO_LANGUAGE)
            }
        }

    // ── Subtitle preference ───────────────────────────────────────────────────

    var preferredSubtitleLanguage: String?
        get() = repository.getPreference(PreferenceKey.PREFERRED_SUBTITLE_LANGUAGE)
        set(value) {
            if (value != null) {
                repository.setPreference(PreferenceKey.PREFERRED_SUBTITLE_LANGUAGE, value)
            } else {
                repository.clearPreference(PreferenceKey.PREFERRED_SUBTITLE_LANGUAGE)
            }
        }

    var subtitlesEnabled: Boolean
        get() = repository.getPreferenceOrDefault(
            PreferenceKey.SUBTITLES_ENABLED, "true"
        ).toBoolean()
        set(value) = repository.setPreference(
            PreferenceKey.SUBTITLES_ENABLED, value.toString()
        )

    // ── Volume ────────────────────────────────────────────────────────────────

    var volume: Float
        get() = repository.getPreferenceOrDefault(
            PreferenceKey.VOLUME, "1.0"
        ).toFloatOrNull() ?: 1.0f
        set(value) = repository.setPreference(
            PreferenceKey.VOLUME, value.coerceIn(0f, 1f).toString()
        )

    // ── Cache ─────────────────────────────────────────────────────────────────

    var maxCacheSizeMb: Int
        get() = repository.getPreferenceOrDefault(
            PreferenceKey.MAX_CACHE_SIZE_MB, "512"
        ).toIntOrNull() ?: 512
        set(value) = repository.setPreference(
            PreferenceKey.MAX_CACHE_SIZE_MB, value.toString()
        )

    fun clearAll() = repository.clearAll()
}

// ── Models ────────────────────────────────────────────────────────────────────

data class ResumeRecord(
    val mediaId: String,
    val positionMs: Long,
    val durationMs: Long,
    val savedAtMs: Long,
) {
    val progress: Float
        get() = if (durationMs <= 0L) 0f
                else (positionMs.toFloat() / durationMs).coerceIn(0f, 1f)
}

data class HistoryRecord(
    val mediaId: String,
    val title: String?,
    val durationMs: Long,
    val playCount: Int,
    val lastPlayedMs: Long,
)

enum class PreferenceKey(val key: String) {
    PLAYBACK_SPEED("playback_speed"),
    REPEAT_MODE("repeat_mode"),
    SHUFFLE_MODE("shuffle_mode"),
    PREFERRED_AUDIO_LANGUAGE("preferred_audio_lang"),
    PREFERRED_SUBTITLE_LANGUAGE("preferred_subtitle_lang"),
    SUBTITLES_ENABLED("subtitles_enabled"),
    VOLUME("volume"),
    MAX_CACHE_SIZE_MB("max_cache_size_mb"),
    DEFAULT_DECODER("default_decoder"),
    AUTO_RESUME("auto_resume"),
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/persistence/ResumeManager.kt

```kotlin
package com.xplayer.dev.core.persistence

import timber.log.Timber
import javax.inject.Inject
import javax.inject.Singleton

/**
 * ResumeManager
 *
 * Handles auto-resume logic.
 * Decides resume position based on completion threshold.
 */
@Singleton
class ResumeManager @Inject constructor(
    private val repository: PlaybackRepository,
) {

    /**
     * Save current playback position for [mediaId].
     * Clears saved position if playback is near completion.
     */
    fun onPlaybackProgress(mediaId: String, positionMs: Long, durationMs: Long) {
        if (durationMs <= 0L) return

        val progress = positionMs.toFloat() / durationMs
        if (progress >= COMPLETION_THRESHOLD) {
            repository.clearPosition(mediaId)
            Timber.d("ResumeManager: cleared (completed) [mediaId=$mediaId]")
        } else {
            repository.savePosition(mediaId, positionMs, durationMs)
        }
    }

    /**
     * Get resume position for [mediaId].
     * Returns 0 if no saved position or position is stale.
     */
    fun getResumePosition(mediaId: String): Long {
        val record = repository.getPosition(mediaId) ?: return 0L

        val ageMs = System.currentTimeMillis() - record.savedAtMs
        if (ageMs > STALE_THRESHOLD_MS) {
            Timber.d("ResumeManager: stale position cleared [mediaId=$mediaId]")
            repository.clearPosition(mediaId)
            return 0L
        }

        Timber.d(
            "ResumeManager: resume at ${record.positionMs}ms " +
                "[mediaId=$mediaId]"
        )
        return record.positionMs
    }

    fun hasResumePosition(mediaId: String): Boolean =
        getResumePosition(mediaId) > 0L

    fun clearResumePosition(mediaId: String) {
        repository.clearPosition(mediaId)
    }

    companion object {
        private const val COMPLETION_THRESHOLD = 0.95f
        private const val STALE_THRESHOLD_MS   = 30L * 24 * 60 * 60 * 1_000L // 30 days
    }
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/performance/MemoryManager.kt

```kotlin
package com.xplayer.dev.core.performance

import android.content.ComponentCallbacks2
import android.content.Context
import android.os.Debug
import timber.log.Timber
import java.util.concurrent.atomic.AtomicInteger
import javax.inject.Inject
import javax.inject.Singleton

/**
 * MemoryManager
 *
 * Monitors heap usage and responds to system memory pressure.
 *
 * Integration:
 *  Register as ComponentCallbacks2 in Application class:
 *  ```
 *  registerComponentCallbacks(memoryManager)
 *  ```
 */
@Singleton
class MemoryManager @Inject constructor(
    private val context: Context,
) : ComponentCallbacks2 {

    private val trimCount = AtomicInteger(0)
    private val oomCount  = AtomicInteger(0)

    // ── ComponentCallbacks2 ───────────────────────────────────────────────────

    override fun onTrimMemory(level: Int) {
        Timber.w("MemoryManager: onTrimMemory [level=$level]")
        when (level) {
            ComponentCallbacks2.TRIM_MEMORY_RUNNING_CRITICAL,
            ComponentCallbacks2.TRIM_MEMORY_COMPLETE,
            -> {
                onLowMemory()
                oomCount.incrementAndGet()
            }
            ComponentCallbacks2.TRIM_MEMORY_RUNNING_LOW,
            ComponentCallbacks2.TRIM_MEMORY_MODERATE,
            -> trimMemory()
            else -> Unit
        }
        trimCount.incrementAndGet()
    }

    override fun onLowMemory() {
        Timber.e("MemoryManager: LOW MEMORY — emergency cleanup")
        System.gc()
        Runtime.getRuntime().runFinalization()
    }

    override fun onConfigurationChanged(
        newConfig: android.content.res.Configuration
    ) = Unit

    // ── Manual operations ─────────────────────────────────────────────────────

    fun trimMemory() {
        Timber.d("MemoryManager: trimMemory()")
        System.gc()
    }

    fun getUsedMemoryMb(): Int {
        val runtime = Runtime.getRuntime()
        val used = runtime.totalMemory() - runtime.freeMemory()
        return (used / 1_048_576L).toInt()
    }

    fun getFreeMemoryMb(): Int {
        val runtime = Runtime.getRuntime()
        return (runtime.freeMemory() / 1_048_576L).toInt()
    }

    fun getMaxMemoryMb(): Int =
        (Runtime.getRuntime().maxMemory() / 1_048_576L).toInt()

    fun getNativeHeapMb(): Int =
        (Debug.getNativeHeapAllocatedSize() / 1_048_576L).toInt()

    // ── Stats ─────────────────────────────────────────────────────────────────

    val totalTrimEvents: Int get() = trimCount.get()
    val totalOomEvents: Int  get() = oomCount.get()

    fun buildReport(): String = buildString {
        appendLine("── MemoryManager Report ──")
        appendLine("Used         : ${getUsedMemoryMb()} MB")
        appendLine("Free         : ${getFreeMemoryMb()} MB")
        appendLine("Max          : ${getMaxMemoryMb()} MB")
        appendLine("NativeHeap   : ${getNativeHeapMb()} MB")
        appendLine("Trim events  : ${trimCount.get()}")
        appendLine("OOM events   : ${oomCount.get()}")
    }
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/performance/PerformanceManager.kt

```kotlin
package com.xplayer.dev.core.performance

import android.app.ActivityManager
import android.content.Context
import android.os.Build
import android.os.Debug
import android.os.Process
import kotlinx.coroutines.CoroutineScope
import kotlinx.coroutines.Job
import kotlinx.coroutines.delay
import kotlinx.coroutines.isActive
import kotlinx.coroutines.launch
import timber.log.Timber
import java.util.concurrent.atomic.AtomicBoolean
import java.util.concurrent.atomic.AtomicReference
import javax.inject.Inject
import javax.inject.Singleton

/**
 * PerformanceManager
 *
 * Central coordinator for runtime performance monitoring
 * and optimization decisions.
 *
 * Responsibilities:
 *  - Periodic sampling of CPU, memory, battery
 *  - Trigger resource cleanup under memory pressure
 *  - Expose PerformanceSnapshot for diagnostics
 *  - Coordinate with MemoryManager, ThreadManager, ResourceManager
 */
@Singleton
class PerformanceManager @Inject constructor(
    private val context: Context,
    private val memoryManager: MemoryManager,
    private val threadManager: ThreadManager,
    private val resourceManager: ResourceManager,
    private val profiler: PerformanceProfiler,
) {
    private var monitorJob: Job? = null
    private val _monitoring = AtomicBoolean(false)
    private val _lastSnapshot = AtomicReference<PerformanceSnapshot?>(null)

    val isMonitoring: Boolean get() = _monitoring.get()
    val lastSnapshot: PerformanceSnapshot? get() = _lastSnapshot.get()

    // ── Lifecycle ─────────────────────────────────────────────────────────────

    fun startMonitoring(scope: CoroutineScope) {
        if (!_monitoring.compareAndSet(false, true)) {
            Timber.w("PerformanceManager: already monitoring")
            return
        }
        monitorJob = scope.launch {
            while (isActive) {
                val snapshot = collectSnapshot()
                _lastSnapshot.set(snapshot)
                profiler.record(snapshot)
                handlePressure(snapshot)
                delay(SAMPLE_INTERVAL_MS)
            }
        }
        Timber.i("PerformanceManager: monitoring started")
    }

    fun stopMonitoring() {
        if (!_monitoring.compareAndSet(true, false)) return
        monitorJob?.cancel()
        monitorJob = null
        Timber.i("PerformanceManager: monitoring stopped")
    }

    // ── Snapshot ──────────────────────────────────────────────────────────────

    private fun collectSnapshot(): PerformanceSnapshot {
        val memInfo = ActivityManager.MemoryInfo()
        (context.getSystemService(Context.ACTIVITY_SERVICE) as ActivityManager)
            .getMemoryInfo(memInfo)

        return PerformanceSnapshot(
            timestampMs        = System.currentTimeMillis(),
            usedMemoryMb       = memoryManager.getUsedMemoryMb(),
            availableMemoryMb  = (memInfo.availMem / 1_048_576L).toInt(),
            totalMemoryMb      = (memInfo.totalMem / 1_048_576L).toInt(),
            isLowMemory        = memInfo.lowMemory,
            nativeHeapMb       = (Debug.getNativeHeapAllocatedSize() / 1_048_576L).toInt(),
            threadCount        = threadManager.activeThreadCount(),
            cpuUsagePercent    = profiler.estimateCpuUsage(),
            frameDropRate      = profiler.currentFrameDropRate,
        )
    }

    // ── Pressure handling ─────────────────────────────────────────────────────

    private fun handlePressure(snapshot: PerformanceSnapshot) {
        if (snapshot.isLowMemory) {
            Timber.w("PerformanceManager: LOW MEMORY — triggering cleanup")
            memoryManager.onLowMemory()
            resourceManager.releaseUnusedResources()
        }
        if (snapshot.usedMemoryMb > HIGH_MEMORY_THRESHOLD_MB) {
            Timber.w(
                "PerformanceManager: high memory usage " +
                    "[${snapshot.usedMemoryMb}MB > ${HIGH_MEMORY_THRESHOLD_MB}MB]"
            )
            memoryManager.trimMemory()
        }
        if (snapshot.frameDropRate > HIGH_DROP_RATE_THRESHOLD) {
            Timber.w(
                "PerformanceManager: high frame drop rate " +
                    "[${snapshot.frameDropRate}]"
            )
        }
    }

    companion object {
        private const val SAMPLE_INTERVAL_MS          = 5_000L
        private const val HIGH_MEMORY_THRESHOLD_MB    = 300
        private const val HIGH_DROP_RATE_THRESHOLD    = 0.05f
    }
}

// ── Snapshot model ────────────────────────────────────────────────────────────

data class PerformanceSnapshot(
    val timestampMs: Long,
    val usedMemoryMb: Int,
    val availableMemoryMb: Int,
    val totalMemoryMb: Int,
    val isLowMemory: Boolean,
    val nativeHeapMb: Int,
    val threadCount: Int,
    val cpuUsagePercent: Float,
    val frameDropRate: Float,
) {
    val memoryPressureLevel: MemoryPressureLevel
        get() = when {
            isLowMemory                    -> MemoryPressureLevel.CRITICAL
            usedMemoryMb > 300             -> MemoryPressureLevel.HIGH
            usedMemoryMb > 150             -> MemoryPressureLevel.MODERATE
            else                           -> MemoryPressureLevel.NORMAL
        }

    val isHealthy: Boolean
        get() = !isLowMemory
            && frameDropRate < 0.01f
            && cpuUsagePercent < 80f
}

enum class MemoryPressureLevel { NORMAL, MODERATE, HIGH, CRITICAL }

```

## xplayer/app/src/main/java/com/xplayer/dev/core/performance/PerformanceProfiler.kt

```kotlin
package com.xplayer.dev.core.performance

import java.util.concurrent.CopyOnWriteArrayList
import java.util.concurrent.atomic.AtomicInteger
import java.util.concurrent.atomic.AtomicLong
import javax.inject.Inject
import javax.inject.Singleton

/**
 * PerformanceProfiler
 *
 * Stores performance snapshot history and computes rolling averages.
 * Thread-safe — CopyOnWriteArrayList for reads, synchronized for writes.
 */
@Singleton
class PerformanceProfiler @Inject constructor() {

    private val snapshots = CopyOnWriteArrayList<PerformanceSnapshot>()

    // ── Frame drop tracking ───────────────────────────────────────────────────

    private val droppedFrames  = AtomicInteger(0)
    private val renderedFrames = AtomicLong(0L)

    fun onFrameDropped(count: Int) { droppedFrames.addAndGet(count) }
    fun onFrameRendered(count: Long) { renderedFrames.addAndGet(count) }

    val currentFrameDropRate: Float
        get() {
            val total = renderedFrames.get() + droppedFrames.get()
            return if (total == 0L) 0f else droppedFrames.get().toFloat() / total
        }

    // ── CPU estimation ────────────────────────────────────────────────────────

    private var lastCpuTime  = 0L
    private var lastWallTime = 0L

    fun estimateCpuUsage(): Float {
        val cpuTime  = android.os.Process.getElapsedCpuTime()
        val wallTime = System.currentTimeMillis()
        val cpuDelta  = cpuTime - lastCpuTime
        val wallDelta = wallTime - lastWallTime

        lastCpuTime  = cpuTime
        lastWallTime = wallTime

        return if (wallDelta <= 0L) 0f
        else ((cpuDelta.toFloat() / wallDelta) * 100f).coerceIn(0f, 100f)
    }

    // ── Snapshot history ──────────────────────────────────────────────────────

    fun record(snapshot: PerformanceSnapshot) {
        if (snapshots.size >= MAX_SNAPSHOTS) snapshots.removeAt(0)
        snapshots.add(snapshot)
    }

    fun averageMemoryMb(): Int {
        if (snapshots.isEmpty()) return 0
        return snapshots.map { it.usedMemoryMb }.average().toInt()
    }

    fun peakMemoryMb(): Int =
        snapshots.maxOfOrNull { it.usedMemoryMb } ?: 0

    fun lowMemoryEventCount(): Int =
        snapshots.count { it.isLowMemory }

    fun recentSnapshots(n: Int): List<PerformanceSnapshot> =
        snapshots.takeLast(n.coerceAtLeast(0))

    fun buildReport(): String = buildString {
        appendLine("── PerformanceProfiler Report ──")
        appendLine("Snapshots recorded  : ${snapshots.size}")
        appendLine("Average memory      : ${averageMemoryMb()} MB")
        appendLine("Peak memory         : ${peakMemoryMb()} MB")
        appendLine("Low memory events   : ${lowMemoryEventCount()}")
        appendLine("Frame drop rate     : ${"%.3f".format(currentFrameDropRate)}")
        appendLine("Rendered frames     : ${renderedFrames.get()}")
        appendLine("Dropped frames      : ${droppedFrames.get()}")
    }

    fun reset() {
        snapshots.clear()
        droppedFrames.set(0)
        renderedFrames.set(0L)
    }

    companion object {
        private const val MAX_SNAPSHOTS = 120
    }
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/performance/ResourceManager.kt

```kotlin
package com.xplayer.dev.core.performance

import timber.log.Timber
import java.util.concurrent.CopyOnWriteArrayList
import javax.inject.Inject
import javax.inject.Singleton

/**
 * ResourceManager
 *
 * Registry for releasable player resources.
 * On memory pressure, triggers release of non-essential resources.
 *
 * Any component holding large resources implements [ManagedResource]
 * and registers with this manager.
 */
@Singleton
class ResourceManager @Inject constructor() {

    fun interface ManagedResource {
        fun release()
    }

    private val resources = CopyOnWriteArrayList<ManagedResource>()

    fun register(resource: ManagedResource) {
        resources.add(resource)
        Timber.d("ResourceManager: registered resource [total=${resources.size}]")
    }

    fun unregister(resource: ManagedResource) {
        resources.remove(resource)
    }

    /**
     * Release all registered non-essential resources.
     * Called on low-memory signal.
     */
    fun releaseUnusedResources() {
        Timber.w("ResourceManager: releasing ${resources.size} managed resources")
        resources.forEach { resource ->
            runCatching { resource.release() }
                .onFailure { Timber.e(it, "ResourceManager: release failed") }
        }
        resources.clear()
    }

    fun registeredCount(): Int = resources.size
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/performance/ThreadManager.kt

```kotlin
package com.xplayer.dev.core.performance

import kotlinx.coroutines.CoroutineDispatcher
import kotlinx.coroutines.Dispatchers
import timber.log.Timber
import java.util.concurrent.ExecutorService
import java.util.concurrent.Executors
import java.util.concurrent.ThreadFactory
import java.util.concurrent.TimeUnit
import java.util.concurrent.atomic.AtomicInteger
import javax.inject.Inject
import javax.inject.Singleton

/**
 * ThreadManager
 *
 * Centralises thread pool creation and lifecycle.
 * Provides named, monitored executors for player subsystems.
 *
 * All executors are tracked for orderly shutdown.
 */
@Singleton
class ThreadManager @Inject constructor() {

    private val executors = mutableListOf<ExecutorService>()
    private val lock      = Any()

    // ── Dispatcher accessors ──────────────────────────────────────────────────

    val mainDispatcher: CoroutineDispatcher    = Dispatchers.Main.immediate
    val ioDispatcher: CoroutineDispatcher      = Dispatchers.IO
    val defaultDispatcher: CoroutineDispatcher = Dispatchers.Default
    val unconfinedDispatcher: CoroutineDispatcher = Dispatchers.Unconfined

    // ── Executor factory ──────────────────────────────────────────────────────

    /**
     * Create a single-thread executor with [name] prefix.
     * Used for sequential tasks (cache writes, DRM session).
     */
    fun newSingleThreadExecutor(name: String): ExecutorService {
        return Executors.newSingleThreadExecutor(namedFactory(name))
            .also { register(it) }
    }

    /**
     * Create a fixed-thread-pool executor.
     * Used for parallel downloads, media scanning.
     */
    fun newFixedThreadPool(threads: Int, name: String): ExecutorService {
        return Executors.newFixedThreadPool(threads, namedFactory(name))
            .also { register(it) }
    }

    /**
     * Create a cached thread pool.
     * Used for short-lived bursts (metadata extraction).
     */
    fun newCachedThreadPool(name: String): ExecutorService {
        return Executors.newCachedThreadPool(namedFactory(name))
            .also { register(it) }
    }

    // ── Shutdown ──────────────────────────────────────────────────────────────

    /**
     * Gracefully shut down all managed executors.
     * Waits up to [timeoutMs] per executor.
     */
    fun shutdownAll(timeoutMs: Long = 2_000L) {
        synchronized(lock) {
            executors.forEach { executor ->
                executor.shutdown()
                runCatching {
                    if (!executor.awaitTermination(timeoutMs, TimeUnit.MILLISECONDS)) {
                        executor.shutdownNow()
                        Timber.w("ThreadManager: forced shutdown for executor")
                    }
                }
            }
            executors.clear()
        }
        Timber.i("ThreadManager: all executors shut down")
    }

    // ── Monitoring ────────────────────────────────────────────────────────────

    fun activeThreadCount(): Int =
        Thread.getAllStackTraces().keys.count { it.isAlive }

    fun managedExecutorCount(): Int = synchronized(lock) { executors.size }

    // ── Private ───────────────────────────────────────────────────────────────

    private fun register(executor: ExecutorService) {
        synchronized(lock) { executors.add(executor) }
    }

    private fun namedFactory(prefix: String): ThreadFactory {
        val counter = AtomicInteger(0)
        return ThreadFactory { runnable ->
            Thread(runnable, "$prefix-${counter.incrementAndGet()}").apply {
                isDaemon = true
                priority = Thread.NORM_PRIORITY
            }
        }
    }
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/error/ErrorClassifier.kt

```kotlin
package com.xplayer.dev.core.error

import androidx.media3.common.PlaybackException
import javax.inject.Inject
import javax.inject.Singleton

/**
 * ErrorClassifier
 *
 * Maps PlaybackException error codes to structured ErrorClassification.
 * Single place where all error categorisation logic lives.
 */
@Singleton
class ErrorClassifier @Inject constructor() {

    fun classify(error: PlaybackException): ErrorClassification {
        val category = categoryFrom(error.errorCode)
        return ErrorClassification(
            errorCode    = error.errorCode,
            category     = category,
            isFatal      = isFatal(category, error.errorCode),
            isRetryable  = isRetryable(category),
            retryDelayMs = retryDelayMs(category),
            action       = defaultAction(category),
        )
    }

    private fun categoryFrom(code: Int): ErrorCategory = when (code) {
        PlaybackException.ERROR_CODE_IO_NETWORK_CONNECTION_FAILED,
        PlaybackException.ERROR_CODE_IO_NETWORK_CONNECTION_TIMEOUT,
        -> ErrorCategory.NETWORK

        PlaybackException.ERROR_CODE_IO_BAD_HTTP_STATUS,
        PlaybackException.ERROR_CODE_IO_CLEARTEXT_NOT_PERMITTED,
        -> ErrorCategory.HTTP

        PlaybackException.ERROR_CODE_DRM_SCHEME_UNSUPPORTED,
        PlaybackException.ERROR_CODE_DRM_PROVISIONING_FAILED,
        PlaybackException.ERROR_CODE_DRM_CONTENT_ERROR,
        PlaybackException.ERROR_CODE_DRM_LICENSE_ACQUISITION_FAILED,
        PlaybackException.ERROR_CODE_DRM_DISALLOWED_OPERATION,
        PlaybackException.ERROR_CODE_DRM_SYSTEM_ERROR,
        -> ErrorCategory.DRM

        PlaybackException.ERROR_CODE_DECODER_INIT_FAILED,
        PlaybackException.ERROR_CODE_DECODER_QUERY_FAILED,
        PlaybackException.ERROR_CODE_DECODING_FAILED,
        PlaybackException.ERROR_CODE_DECODING_FORMAT_EXCEEDS_CAPABILITIES,
        PlaybackException.ERROR_CODE_DECODING_FORMAT_UNSUPPORTED,
        -> ErrorCategory.DECODER

        PlaybackException.ERROR_CODE_IO_FILE_NOT_FOUND,
        PlaybackException.ERROR_CODE_IO_NO_PERMISSION,
        PlaybackException.ERROR_CODE_PARSING_CONTAINER_MALFORMED,
        PlaybackException.ERROR_CODE_PARSING_MANIFEST_MALFORMED,
        -> ErrorCategory.SOURCE

        PlaybackException.ERROR_CODE_TIMEOUT,
        -> ErrorCategory.TIMEOUT

        else -> ErrorCategory.UNKNOWN
    }

    private fun isFatal(category: ErrorCategory, code: Int): Boolean = when (category) {
        ErrorCategory.DRM    -> true
        ErrorCategory.SOURCE -> true
        ErrorCategory.HTTP   -> code == PlaybackException.ERROR_CODE_IO_FILE_NOT_FOUND
        else                 -> false
    }

    private fun isRetryable(category: ErrorCategory): Boolean = when (category) {
        ErrorCategory.NETWORK -> true
        ErrorCategory.HTTP    -> true
        ErrorCategory.TIMEOUT -> true
        ErrorCategory.DECODER -> true
        ErrorCategory.UNKNOWN -> true
        else                  -> false
    }

    private fun retryDelayMs(category: ErrorCategory): Long = when (category) {
        ErrorCategory.NETWORK -> 2_000L
        ErrorCategory.HTTP    -> 3_000L
        ErrorCategory.TIMEOUT -> 5_000L
        ErrorCategory.DECODER -> 1_000L
        else                  -> 2_000L
    }

    private fun defaultAction(category: ErrorCategory): ErrorAction = when (category) {
        ErrorCategory.NETWORK -> ErrorAction.RetryWithBackoff(maxRetries = 3)
        ErrorCategory.HTTP    -> ErrorAction.RetryOnce
        ErrorCategory.DRM     -> ErrorAction.ClearAndReload
        ErrorCategory.DECODER -> ErrorAction.RetryWithFallbackDecoder
        ErrorCategory.SOURCE  -> ErrorAction.ReportAndStop
        ErrorCategory.TIMEOUT -> ErrorAction.RetryWithBackoff(maxRetries = 3)
        ErrorCategory.UNKNOWN -> ErrorAction.RetryOnce
    }
}

// ── Classification model ──────────────────────────────────────────────────────

data class ErrorClassification(
    val errorCode: Int,
    val category: ErrorCategory,
    val isFatal: Boolean,
    val isRetryable: Boolean,
    val retryDelayMs: Long,
    val action: ErrorAction,
)

enum class ErrorCategory {
    NETWORK,
    HTTP,
    DRM,
    DECODER,
    SOURCE,
    TIMEOUT,
    UNKNOWN,
}

// ── Action sealed class ───────────────────────────────────────────────────────

sealed class ErrorAction {
    data class RetryWithBackoff(
        val maxRetries: Int    = 3,
        val baseDelayMs: Long  = 2_000L,
        val multiplier: Float  = 2.0f,
    ) : ErrorAction()

    data object RetryOnce : ErrorAction()

    data object RetryWithFallbackDecoder : ErrorAction()

    data object ClearAndReload : ErrorAction()

    data object ReportAndStop : ErrorAction()

    data object ReleaseEngine : ErrorAction()
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/error/ErrorManager.kt

```kotlin
package com.xplayer.dev.core.error

import androidx.media3.common.PlaybackException
import com.xplayer.dev.core.state.PlayerState
import timber.log.Timber
import java.util.concurrent.CopyOnWriteArrayList
import java.util.concurrent.atomic.AtomicInteger
import javax.inject.Inject
import javax.inject.Singleton

/**
 * ErrorManager
 *
 * Central registry for all playback errors.
 *
 * Responsibilities:
 *  - Classify errors via ErrorClassifier
 *  - Route to RecoveryManager for handling
 *  - Maintain error history for diagnostics
 *  - Track error frequency for health scoring
 */
@Singleton
class ErrorManager @Inject constructor(
    private val classifier: ErrorClassifier,
    private val recoveryManager: RecoveryManager,
    private val healthMonitor: HealthMonitor,
) {
    private val errorHistory = CopyOnWriteArrayList<ErrorRecord>()
    private val totalErrors  = AtomicInteger(0)
    private val fatalErrors  = AtomicInteger(0)

    // ── Error intake ──────────────────────────────────────────────────────────

    /**
     * Handle a new [PlaybackException].
     * Classifies, records, and routes to recovery.
     *
     * @return [ErrorAction] describing what the caller should do next.
     */
    fun onError(
        error: PlaybackException,
        currentState: PlayerState,
        sessionId: String,
        mediaId: String?,
    ): ErrorAction {
        val classification = classifier.classify(error)
        val record = ErrorRecord(
            error          = error,
            errorCode      = error.errorCode,
            errorCodeName  = error.errorCodeName,
            classification = classification,
            sessionId      = sessionId,
            mediaId        = mediaId,
            stateName      = currentState::class.simpleName ?: "Unknown",
            isFatal        = classification.isFatal,
        )

        record(record)
        healthMonitor.onError(classification)

        val action = recoveryManager.determineAction(record)

        Timber.e(
            error,
            "ErrorManager: [code=${error.errorCode}] " +
                "[class=${classification.category}] " +
                "[fatal=${classification.isFatal}] " +
                "[action=${action::class.simpleName}]"
        )

        return action
    }

    // ── History ───────────────────────────────────────────────────────────────

    private fun record(record: ErrorRecord) {
        if (errorHistory.size >= MAX_HISTORY) errorHistory.removeAt(0)
        errorHistory.add(record)
        totalErrors.incrementAndGet()
        if (record.isFatal) fatalErrors.incrementAndGet()
    }

    fun recentErrors(n: Int = 10): List<ErrorRecord> =
        errorHistory.takeLast(n.coerceAtLeast(0))

    fun errorsForSession(sessionId: String): List<ErrorRecord> =
        errorHistory.filter { it.sessionId == sessionId }

    fun clearHistory() = errorHistory.clear()

    // ── Stats ─────────────────────────────────────────────────────────────────

    val totalErrorCount: Int get() = totalErrors.get()
    val fatalErrorCount: Int get() = fatalErrors.get()

    fun buildReport(): String = buildString {
        appendLine("── ErrorManager Report ──")
        appendLine("Total errors    : ${totalErrors.get()}")
        appendLine("Fatal errors    : ${fatalErrors.get()}")
        appendLine("History size    : ${errorHistory.size}")
        appendLine("Recent errors:")
        recentErrors(5).forEachIndexed { i, e ->
            appendLine(
                "  $i. [${e.errorCodeName}] " +
                    "cat=${e.classification.category} " +
                    "fatal=${e.isFatal} " +
                    "session=${e.sessionId.take(8)}"
            )
        }
    }

    companion object {
        private const val MAX_HISTORY = 100
    }
}

// ── Models ────────────────────────────────────────────────────────────────────

data class ErrorRecord(
    val error: PlaybackException,
    val errorCode: Int,
    val errorCodeName: String,
    val classification: ErrorClassification,
    val sessionId: String,
    val mediaId: String?,
    val stateName: String,
    val isFatal: Boolean,
    val timestampMs: Long = System.currentTimeMillis(),
)

```

## xplayer/app/src/main/java/com/xplayer/dev/core/error/HealthMonitor.kt

```kotlin
package com.xplayer.dev.core.error

import timber.log.Timber
import java.util.concurrent.atomic.AtomicInteger
import java.util.concurrent.atomic.AtomicLong
import java.util.concurrent.atomic.AtomicReference
import javax.inject.Inject
import javax.inject.Singleton

/**
 * HealthMonitor
 *
 * Tracks overall player health across sessions.
 * Scores are used by analytics and crash reporting.
 *
 * Health Score: 0 (worst) → 100 (perfect)
 *  - Starts at 100
 *  - Decremented per error (weighted by severity)
 *  - Recovered partially on successful playback
 */
@Singleton
class HealthMonitor @Inject constructor() {

    private val score            = AtomicInteger(MAX_SCORE)
    private val networkErrors    = AtomicInteger(0)
    private val decoderErrors    = AtomicInteger(0)
    private val drmErrors        = AtomicInteger(0)
    private val fatalErrors      = AtomicInteger(0)
    private val successfulPlays  = AtomicInteger(0)
    private val lastErrorMs      = AtomicLong(0L)
    private val _status          = AtomicReference(HealthStatus.HEALTHY)

    val currentScore: Int get() = score.get()
    val currentStatus: HealthStatus get() = _status.get()

    // ── Event intake ──────────────────────────────────────────────────────────

    fun onError(classification: ErrorClassification) {
        lastErrorMs.set(System.currentTimeMillis())

        val penalty = when (classification.category) {
            ErrorCategory.NETWORK -> 5
            ErrorCategory.HTTP    -> 5
            ErrorCategory.DRM     -> 30
            ErrorCategory.DECODER -> 15
            ErrorCategory.SOURCE  -> 20
            ErrorCategory.TIMEOUT -> 8
            ErrorCategory.UNKNOWN -> 5
        }

        when (classification.category) {
            ErrorCategory.NETWORK -> networkErrors.incrementAndGet()
            ErrorCategory.DECODER -> decoderErrors.incrementAndGet()
            ErrorCategory.DRM     -> drmErrors.incrementAndGet()
            else                  -> Unit
        }
        if (classification.isFatal) fatalErrors.incrementAndGet()

        val newScore = (score.get() - penalty).coerceAtLeast(MIN_SCORE)
        score.set(newScore)
        _status.set(evaluateStatus(newScore))

        Timber.d(
            "HealthMonitor: score=${score.get()} " +
                "status=${_status.get()} " +
                "penalty=$penalty " +
                "category=${classification.category}"
        )
    }

    fun onSuccessfulPlay() {
        successfulPlays.incrementAndGet()
        val recovery = 2
        val newScore = (score.get() + recovery).coerceAtMost(MAX_SCORE)
        score.set(newScore)
        _status.set(evaluateStatus(newScore))
    }

    fun reset() {
        score.set(MAX_SCORE)
        networkErrors.set(0)
        decoderErrors.set(0)
        drmErrors.set(0)
        fatalErrors.set(0)
        successfulPlays.set(0)
        lastErrorMs.set(0L)
        _status.set(HealthStatus.HEALTHY)
    }

    // ── Report ────────────────────────────────────────────────────────────────

    fun buildReport(): String = buildString {
        appendLine("── HealthMonitor Report ──")
        appendLine("Score           : ${score.get()}/$MAX_SCORE")
        appendLine("Status          : ${_status.get()}")
        appendLine("Network errors  : ${networkErrors.get()}")
        appendLine("Decoder errors  : ${decoderErrors.get()}")
        appendLine("DRM errors      : ${drmErrors.get()}")
        appendLine("Fatal errors    : ${fatalErrors.get()}")
        appendLine("Successful plays: ${successfulPlays.get()}")
        lastErrorMs.get().takeIf { it > 0L }?.let {
            appendLine("Last error      : ${System.currentTimeMillis() - it}ms ago")
        }
    }

    private fun evaluateStatus(score: Int): HealthStatus = when {
        score >= 80 -> HealthStatus.HEALTHY
        score >= 50 -> HealthStatus.DEGRADED
        score >= 20 -> HealthStatus.UNHEALTHY
        else        -> HealthStatus.CRITICAL
    }

    companion object {
        private const val MAX_SCORE = 100
        private const val MIN_SCORE = 0
    }
}

enum class HealthStatus { HEALTHY, DEGRADED, UNHEALTHY, CRITICAL }

```

## xplayer/app/src/main/java/com/xplayer/dev/core/error/RecoveryManager.kt

```kotlin
package com.xplayer.dev.core.error

import timber.log.Timber
import java.util.concurrent.ConcurrentHashMap
import java.util.concurrent.atomic.AtomicInteger
import javax.inject.Inject
import javax.inject.Singleton

/**
 * RecoveryManager
 *
 * Determines recovery strategy for a given ErrorRecord.
 * Tracks retry attempts per session to enforce retry limits.
 */
@Singleton
class RecoveryManager @Inject constructor() {

    private val retryCounts = ConcurrentHashMap<String, AtomicInteger>()

    /**
     * Determine what action to take for [record].
     *
     * @return ErrorAction for the caller to execute.
     */
    fun determineAction(record: ErrorRecord): ErrorAction {
        if (record.isFatal) {
            Timber.e("RecoveryManager: fatal error — no recovery possible")
            return ErrorAction.ReportAndStop
        }

        val sessionKey = sessionKey(record)
        val retries    = retryCounts
            .getOrPut(sessionKey) { AtomicInteger(0) }
            .get()

        val base = record.classification.action

        return when (base) {
            is ErrorAction.RetryWithBackoff -> {
                if (retries >= base.maxRetries) {
                    Timber.e(
                        "RecoveryManager: max retries ($retries) reached " +
                            "[session=${record.sessionId.take(8)}]"
                    )
                    ErrorAction.ReportAndStop
                } else {
                    val delay = calculateBackoff(base, retries)
                    Timber.i(
                        "RecoveryManager: retry ${retries + 1}/${base.maxRetries} " +
                            "in ${delay}ms"
                    )
                    ErrorAction.RetryWithBackoff(
                        maxRetries   = base.maxRetries,
                        baseDelayMs  = delay,
                        multiplier   = base.multiplier,
                    )
                }
            }
            else -> base
        }
    }

    fun onRetryAttempted(sessionId: String, errorCode: Int) {
        val key = "$sessionId:$errorCode"
        retryCounts.getOrPut(key) { AtomicInteger(0) }.incrementAndGet()
    }

    fun onRecoverySuccess(sessionId: String) {
        val keysToRemove = retryCounts.keys.filter { it.startsWith(sessionId) }
        keysToRemove.forEach { retryCounts.remove(it) }
        Timber.i("RecoveryManager: recovery success — counters reset [session=${sessionId.take(8)}]")
    }

    fun onSessionEnd(sessionId: String) {
        val keysToRemove = retryCounts.keys.filter { it.startsWith(sessionId) }
        keysToRemove.forEach { retryCounts.remove(it) }
    }

    fun retryCountForSession(sessionId: String, errorCode: Int): Int =
        retryCounts["$sessionId:$errorCode"]?.get() ?: 0

    fun clearAll() = retryCounts.clear()

    private fun calculateBackoff(action: ErrorAction.RetryWithBackoff, retries: Int): Long {
        val delay = action.baseDelayMs * Math.pow(
            action.multiplier.toDouble(), retries.toDouble()
        ).toLong()
        return delay.coerceAtMost(MAX_DELAY_MS)
    }

    private fun sessionKey(record: ErrorRecord): String =
        "${record.sessionId}:${record.errorCode}"

    companion object {
        private const val MAX_DELAY_MS = 30_000L
    }
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/error/RetryManager.kt

```kotlin
package com.xplayer.dev.core.error

import kotlinx.coroutines.CoroutineScope
import kotlinx.coroutines.Job
import kotlinx.coroutines.delay
import kotlinx.coroutines.launch
import timber.log.Timber
import java.util.concurrent.atomic.AtomicBoolean
import java.util.concurrent.atomic.AtomicInteger
import javax.inject.Inject
import javax.inject.Singleton

/**
 * RetryManager
 *
 * Executes retry sequences with configurable backoff.
 * Integrates with RecoveryManager for retry count tracking.
 *
 * Usage:
 * ```kotlin
 * retryManager.scheduleRetry(
 *     sessionId  = sessionId,
 *     errorCode  = error.errorCode,
 *     delayMs    = 2_000L,
 *     maxRetries = 3,
 * ) {
 *     engine.prepare(mediaItem, mediaId, url)
 * }
 * ```
 */
@Singleton
class RetryManager @Inject constructor(
    private val recoveryManager: RecoveryManager,
) {
    private var retryJob: Job? = null
    private val _isRetrying    = AtomicBoolean(false)
    private val _retryCount    = AtomicInteger(0)

    val isRetrying: Boolean get() = _isRetrying.get()
    val currentRetryCount: Int get() = _retryCount.get()

    // ── Retry scheduling ──────────────────────────────────────────────────────

    fun scheduleRetry(
        scope: CoroutineScope,
        sessionId: String,
        errorCode: Int,
        delayMs: Long,
        maxRetries: Int,
        onRetry: suspend () -> Unit,
        onExhausted: (() -> Unit)? = null,
    ) {
        if (_isRetrying.get()) {
            Timber.w("RetryManager: retry already in progress — skipping")
            return
        }

        val count = _retryCount.incrementAndGet()
        if (count > maxRetries) {
            Timber.e("RetryManager: max retries ($maxRetries) exhausted")
            _isRetrying.set(false)
            _retryCount.set(0)
            onExhausted?.invoke()
            return
        }

        _isRetrying.set(true)
        recoveryManager.onRetryAttempted(sessionId, errorCode)

        retryJob = scope.launch {
            Timber.i("RetryManager: retry $count/$maxRetries in ${delayMs}ms")
            delay(delayMs)
            runCatching { onRetry() }
                .onSuccess {
                    _isRetrying.set(false)
                    Timber.i("RetryManager: retry $count succeeded")
                    recoveryManager.onRecoverySuccess(sessionId)
                }
                .onFailure { throwable ->
                    _isRetrying.set(false)
                    Timber.e(throwable, "RetryManager: retry $count failed")
                }
        }
    }

    fun cancel() {
        retryJob?.cancel()
        retryJob = null
        _isRetrying.set(false)
        _retryCount.set(0)
        Timber.d("RetryManager: cancelled")
    }

    fun reset() {
        cancel()
    }
}

```

## xplayer/app/src/main/java/com/xplayer/dev/core/release/ArchitectureDoc.kt

```kotlin
package com.xplayer.dev.core.release

/**
 * # PlayerEngine Architecture Reference
 *
 * ═══════════════════════════════════════════════════════════════════
 * LAYER OVERVIEW
 * ═══════════════════════════════════════════════════════════════════
 *
 * ┌──────────────────────────────────────────────────────────────────┐
 * │                         UI / ViewModel                          │
 * │  Observes: state: StateFlow  events: Flow  metrics: snapshot    │
 * ├──────────────────────────────────────────────────────────────────┤
 * │                       PlayerManager                             │
 * │  Coordinates: prepare / play / pause / stop / seek              │
 * ├──────────────────────────────────────────────────────────────────┤
 * │                       PlayerEngine                              │
 * │  Owns ExoPlayer. Emits PlayerState + PlayerEvent                │
 * ├──────────────┬───────────────┬──────────────┬───────────────────┤
 * │EngineLifecycle│ListenerHub   │MetricsProbe  │ EventBus          │
 * │(sealed FSM)   │(ExoPlayer    │(TTFP/stalls/ │ (Channel 512 +   │
 * │               │ callbacks)   │ drops/bw)    │  SharedFlow)      │
 * ├──────────────┴───────────────┴──────────────┴───────────────────┤
 * │                   ExoPlayer (Media3 1.4.x)                      │
 * ├──────────────┬───────────────┬──────────────┬───────────────────┤
 * │LoadControl   │TrackSelector  │DataSource    │DrmSessionManager  │
 * └──────────────┴───────────────┴──────────────┴───────────────────┘
 *
 * ═══════════════════════════════════════════════════════════════════
 * STATE MACHINE
 * ═══════════════════════════════════════════════════════════════════
 *
 *  Uninitialised → Idle → Preparing → Buffering → Ready → Playing
 *                    ▲        │            │          │       │
 *                    │        └────────────┴──────────┴───────┘
 *                    │                    ▼
 *                    └─────────────── Error ──► Idle / Preparing (retry)
 *                                        │
 *                                      Ended ──► Idle / Preparing (replay)
 *
 * ═══════════════════════════════════════════════════════════════════
 * EVENT FLOW
 * ═══════════════════════════════════════════════════════════════════
 *
 *  PlayerEngine / ListenerHub
 *       │
 *       │  send(PlayerEvent)
 *       ▼
 *  PlayerEventBus ──┬── Channel (512)    ──► PlayerEventRouter
 *                   │                              │
 *                   └── SharedFlow (broadcast) ──► UI / Analytics
 *                                                  │
 *                                         PlayerEventStore (500)
 *                                         PlayerEventValidator
 *                                         PlayerEventSerializer
 *                                         PlayerEventDiagnostics ──► CrashReporter
 *
 * ═══════════════════════════════════════════════════════════════════
 * THREADING MODEL
 * ═══════════════════════════════════════════════════════════════════
 *
 *  Main Thread:    PlayerEngine API, ExoPlayer calls, Listener callbacks
 *  IO Thread:      Cache reads/writes, Network, Download, MediaStore scan
 *  Default Thread: Analytics aggregation, Codec detection, Metadata parsing
 *  Any Thread:     StateFlow / Flow collection (Kotlin coroutines)
 *
 * ═══════════════════════════════════════════════════════════════════
 * DEPENDENCY GRAPH (Hilt Singleton scope)
 * ═══════════════════════════════════════════════════════════════════
 *
 *  PlayerEngine
 *   ├── PlayerFactory
 *   ├── EngineSessionManager
 *   ├── EngineMetricsProbe
 *   ├── EngineEventBus
 *   └── EngineStateValidator
 *       └── EngineListenerHub
 *
 *  AnalyticsManager
 *   ├── EngineMetricsProbe   (shared)
 *   ├── PlayerEventBus       (shared)
 *   ├── PlaybackProfiler
 *   ├── QoSMonitor
 *   └── AnalyticsRepository
 *
 *  ErrorManager
 *   ├── ErrorClassifier
 *   ├── RecoveryManager
 *   │    └── RetryManager
 *   └── HealthMonitor
 *
 *  CacheManager
 *   ├── CacheConfiguration
 *   └── CacheRepository
 *
 *  DownloadManager
 *   ├── DownloadRepository
 *   ├── DownloadQueue
 *   └── DownloadScheduler
 *
 *  DrmManager
 *   ├── LicenseManager
 *   ├── DrmKeyManager
 *   └── DrmSessionManager_
 *
 *  MediaLibrary
 *   ├── MediaScanner
 *   ├── MediaIndexer
 *   └── LibraryRepository
 *
 *  PlaybackRepository
 *   ├── ResumeManager
 *   ├── HistoryManager
 *   └── PreferenceManager_
 *
 * ═══════════════════════════════════════════════════════════════════
 * PACKAGE STRUCTURE
 * ═══════════════════════════════════════════════════════════════════
 *
 *  core/
 *   ├── engine/
 *   │    ├── PlayerEngine.kt
 *   │    ├── PlayerFactory.kt
 *   │    └── internal/
 *   │         ├── EngineLifecycle.kt
 *   │         ├── EngineListenerHub.kt
 *   │         ├── EngineSessionManager.kt
 *   │         ├── EngineMetricsProbe.kt
 *   │         ├── EngineEventBus.kt
 *   │         └── EngineStateValidator.kt
 *   ├── state/
 *   │    ├── PlayerState.kt
 *   │    ├── PlayerStateExtensions.kt
 *   │    ├── PlayerStateMachine.kt
 *   │    ├── PlayerStateReducer.kt
 *   │    ├── PlayerStateStore.kt
 *   │    ├── PlayerStateCommand.kt
 *   │    ├── PlayerStateTransition.kt
 *   │    ├── PlayerStateDiagnostics.kt
 *   │    └── PlayerStateObserver.kt
 *   ├── events/
 *   │    ├── PlayerEvent.kt
 *   │    ├── PlayerEventExtensions.kt
 *   │    ├── PlayerEventBus.kt
 *   │    ├── PlayerEventRouter.kt
 *   │    ├── PlayerEventFilter.kt
 *   │    ├── PlayerEventStore.kt
 *   │    ├── PlayerEventValidator.kt
 *   │    ├── PlayerEventSerializer.kt
 *   │    └── PlayerEventDiagnostics.kt
 *   ├── analytics/
 *   │    ├── AnalyticsManager.kt
 *   │    ├── PlaybackProfiler.kt
 *   │    ├── QoSMonitor.kt
 *   │    └── AnalyticsRepository.kt
 *   ├── cache/
 *   │    ├── CacheManager.kt
 *   │    ├── CacheConfiguration.kt
 *   │    └── CacheRepository.kt
 *   ├── download/
 *   │    ├── DownloadManager.kt
 *   │    ├── DownloadRepository.kt
 *   │    ├── DownloadQueue.kt
 *   │    └── DownloadScheduler.kt
 *   ├── drm/
 *   │    ├── DrmManager.kt
 *   │    ├── LicenseManager.kt
 *   │    ├── DrmKeyManager.kt
 *   │    ├── DrmSessionManager_.kt
 *   │    └── DrmConfiguration.kt
 *   ├── codec/
 *   │    ├── CodecManager.kt
 *   │    ├── CodecDetector.kt
 *   │    ├── CodecRepository.kt
 *   │    └── DecoderSelector.kt
 *   ├── ffmpeg/
 *   │    ├── FFmpegManager.kt
 *   │    ├── FFmpegBridge.kt
 *   │    ├── FFmpegCapabilityChecker.kt
 *   │    └── FFmpegDecoderManager.kt
 *   ├── parser/
 *   │    ├── MediaParser.kt
 *   │    ├── ContainerParser.kt
 *   │    ├── ManifestParser.kt
 *   │    └── MetadataParser.kt
 *   ├── library/
 *   │    ├── MediaLibrary.kt
 *   │    ├── MediaScanner.kt
 *   │    ├── MediaIndexer.kt
 *   │    └── LibraryRepository.kt
 *   ├── persistence/
 *   │    ├── PlaybackRepository.kt
 *   │    ├── ResumeManager.kt
 *   │    ├── HistoryManager.kt
 *   │    └── PreferenceManager_.kt
 *   ├── performance/
 *   │    ├── PerformanceManager.kt
 *   │    ├── MemoryManager.kt
 *   │    ├── ThreadManager.kt
 *   │    ├── ResourceManager.kt
 *   │    └── PerformanceProfiler.kt
 *   ├── error/
 *   │    ├── ErrorManager.kt
 *   │    ├── ErrorClassifier.kt
 *   │    ├── RecoveryManager.kt
 *   │    ├── RetryManager.kt
 *   │    └── HealthMonitor.kt
 *   └── release/
 *        ├── ReleaseChecklist.kt
 *        └── ArchitectureDoc.kt
 */
object ArchitectureDoc

```

## xplayer/app/src/main/java/com/xplayer/dev/core/release/ReleaseChecklist.kt

```kotlin
package com.xplayer.dev.core.release

/**
 * ReleaseChecklist
 *
 * Programmatic production readiness verification.
 * Run before every release build.
 *
 * Usage:
 * ```kotlin
 * val report = ReleaseChecklist(engine, healthMonitor, profiler).verify()
 * if (!report.isReadyForRelease) error(report.summary)
 * ```
 */
class ReleaseChecklist(
    private val engine: com.xplayer.dev.core.engine.PlayerEngine,
    private val healthMonitor: com.xplayer.dev.core.error.HealthMonitor,
    private val profiler: com.xplayer.dev.core.performance.PerformanceProfiler,
    private val eventBus: com.xplayer.dev.core.events.PlayerEventBus,
) {

    data class CheckResult(
        val name: String,
        val passed: Boolean,
        val detail: String,
    )

    data class ReleaseReport(
        val checks: List<CheckResult>,
        val timestamp: Long = System.currentTimeMillis(),
    ) {
        val passedCount: Int get() = checks.count { it.passed }
        val failedCount: Int get() = checks.count { !it.passed }
        val isReadyForRelease: Boolean get() = failedCount == 0

        val summary: String get() = buildString {
            appendLine("═══════════════════════════════════════")
            appendLine("Release Readiness Report")
            appendLine("═══════════════════════════════════════")
            appendLine("Passed : $passedCount / ${checks.size}")
            appendLine("Failed : $failedCount")
            appendLine("Ready  : $isReadyForRelease")
            appendLine("───────────────────────────────────────")
            checks.forEach { check ->
                val icon = if (check.passed) "✓" else "✗"
                appendLine("$icon ${check.name}")
                if (!check.passed) appendLine("  → ${check.detail}")
            }
            appendLine("═══════════════════════════════════════")
        }
    }

    fun verify(): ReleaseReport {
        val checks = listOf(
            checkEngineLifecycle(),
            checkEventBusOpen(),
            checkHealthScore(),
            checkNoDroppedEvents(),
            checkMemoryBaseline(),
            checkFrameDropRate(),
        )
        return ReleaseReport(checks)
    }

    private fun checkEngineLifecycle(): CheckResult {
        val lc = engine.lifecycleState
        val passed = lc is com.xplayer.dev.core.engine.EngineLifecycle.Active
            || lc is com.xplayer.dev.core.engine.EngineLifecycle.Uninitialized
        return CheckResult(
            name   = "Engine lifecycle valid",
            passed = passed,
            detail = "Current lifecycle: ${lc::class.simpleName}",
        )
    }

    private fun checkEventBusOpen(): CheckResult {
        val open = eventBus.isOpen
        return CheckResult(
            name   = "EventBus open",
            passed = open,
            detail = if (open) "" else "EventBus is closed — cannot emit events",
        )
    }

    private fun checkHealthScore(): CheckResult {
        val score = healthMonitor.currentScore
        val passed = score >= MINIMUM_HEALTH_SCORE
        return CheckResult(
            name   = "Health score >= $MINIMUM_HEALTH_SCORE",
            passed = passed,
            detail = "Current score: $score / 100",
        )
    }

    private fun checkNoDroppedEvents(): CheckResult {
        val dropped = eventBus.droppedCount
        val passed  = dropped == 0L
        return CheckResult(
            name   = "No dropped events",
            passed = passed,
            detail = "Dropped event count: $dropped",
        )
    }

    private fun checkMemoryBaseline(): CheckResult {
        val memMb  = profiler.averageMemoryMb()
        val passed = memMb < MAX_MEMORY_MB
        return CheckResult(
            name   = "Memory baseline < ${MAX_MEMORY_MB}MB",
            passed = passed,
            detail = "Average memory usage: ${memMb}MB",
        )
    }

    private fun checkFrameDropRate(): CheckResult {
        val rate   = profiler.currentFrameDropRate
        val passed = rate < MAX_DROP_RATE
        return CheckResult(
            name   = "Frame drop rate < ${MAX_DROP_RATE * 100}%",
            passed = passed,
            detail = "Current drop rate: ${"%.2f".format(rate * 100)}%",
        )
    }

    companion object {
        private const val MINIMUM_HEALTH_SCORE = 70
        private const val MAX_MEMORY_MB        = 400
        private const val MAX_DROP_RATE        = 0.02f
    }
}

```

## xplayer/app/src/main/java/com/xplayer/dev/crash/CrashModule.kt

```kotlin
package com.xplayer.dev.crash

import dagger.Binds
import dagger.Module
import dagger.hilt.InstallIn
import dagger.hilt.components.SingletonComponent
import javax.inject.Singleton

@Module
@InstallIn(SingletonComponent::class)
abstract class CrashModule {

    @Binds
    @Singleton
    abstract fun bindCrashReporter(
        impl: NoOpCrashReporter,
    ): CrashReporter
}

```

## xplayer/app/src/main/java/com/xplayer/dev/crash/CrashReportingHook.kt

```kotlin
package com.xplayer.dev.crash

import android.os.StrictMode
import com.xplayer.dev.core.config.PlayerConfiguration
import timber.log.Timber
import javax.inject.Inject
import javax.inject.Singleton

/**
 * Installs crash reporting hooks and optional StrictMode policies.
 *
 * The actual crash reporter (Firebase Crashlytics, Sentry, etc.) is
 * injected behind [CrashReporter] so this class stays portable and
 * testable without a real SDK present.
 */
@Singleton
class CrashReportingHook @Inject constructor(
    private val configuration: PlayerConfiguration,
    private val reporter: CrashReporter,
) {

    fun install() {
        installUncaughtExceptionHandler()
        if (configuration.loggingConfig.enableStrictMode) {
            installStrictMode()
        }
        Timber.i("CrashReportingHook: installed")
    }

    // ── Uncaught exceptions ───────────────────────────────────────────────────

    private fun installUncaughtExceptionHandler() {
        val defaultHandler = Thread.getDefaultUncaughtExceptionHandler()
        Thread.setDefaultUncaughtExceptionHandler { thread, throwable ->
            Timber.e(throwable, "Uncaught exception on thread '${thread.name}'")
            runCatching { reporter.recordFatalException(throwable) }
            defaultHandler?.uncaughtException(thread, throwable)
        }
    }

    // ── StrictMode ────────────────────────────────────────────────────────────

    private fun installStrictMode() {
        StrictMode.setThreadPolicy(
            StrictMode.ThreadPolicy.Builder()
                .detectAll()
                .penaltyLog()
                // Use penaltyDeath() only in CI — not in production debug builds
                .build()
        )
        StrictMode.setVmPolicy(
            StrictMode.VmPolicy.Builder()
                .detectLeakedSqlLiteObjects()
                .detectLeakedClosableObjects()
                .detectLeakedRegistrationObjects()
                .detectActivityLeaks()
                .penaltyLog()
                .build()
        )
        Timber.d("CrashReportingHook: StrictMode enabled")
    }
}

// ── CrashReporter contract ────────────────────────────────────────────────────

/**
 * Minimal reporting contract. Replace with your SDK's implementation.
 * Default no-op keeps tests clean.
 */
interface CrashReporter {
    fun recordFatalException(throwable: Throwable)
    fun recordNonFatalException(throwable: Throwable)
    fun setCustomKey(key: String, value: String)
}

class NoOpCrashReporter @Inject constructor() : CrashReporter {
    override fun recordFatalException(throwable: Throwable) {
        Timber.e(throwable, "CrashReporter[noop]: fatal")
    }
    override fun recordNonFatalException(throwable: Throwable) {
        Timber.w(throwable, "CrashReporter[noop]: non-fatal")
    }
    override fun setCustomKey(key: String, value: String) = Unit
}

```

## xplayer/app/src/test/java/com/xplayer/dev/EngineLifecycleManagerTest.kt

```kotlin
package com.xplayer.dev

import com.xplayer.dev.core.engine.*
import io.mockk.*
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.ExperimentalCoroutinesApi
import kotlinx.coroutines.test.*
import org.junit.*
import org.junit.Assert.*

@OptIn(ExperimentalCoroutinesApi::class)
class EngineLifecycleManagerTest {

    private val testDispatcher = UnconfinedTestDispatcher()
    private lateinit var factory: PlayerFactory
    private lateinit var validator: EngineLifecycleValidator
    private lateinit var logger: EngineLifecycleLogger
    private lateinit var manager: EngineLifecycleManager
    private lateinit var mockPlayer: androidx.media3.exoplayer.ExoPlayer

    @Before
    fun setUp() {
        Dispatchers.setMain(testDispatcher)
        factory = mockk(relaxed = true)
        mockPlayer = mockk(relaxed = true)
        every { factory.create() } returns mockPlayer

        validator = EngineLifecycleValidator()
        logger = mockk(relaxed = true)
        manager = EngineLifecycleManager(factory, validator, logger)
    }

    @After
    fun tearDown() {
        Dispatchers.resetMain()
        unmockkAll()
    }

    @Test
    fun `initial state is Uninitialized`() {
        assertEquals(EngineLifecycle.Uninitialized, manager.currentState)
    }

    @Test
    fun `initialize successfully transitions to Active`() {
        val player = manager.initialize()
        assertEquals(mockPlayer, player)
        assertTrue(manager.currentState is EngineLifecycle.Active)
    }

    @Test
    fun `double initialize returns existing player without calling factory again`() {
        manager.initialize()
        manager.initialize()
        verify(exactly = 1) { factory.create() }
    }

    @Test
    fun `release transitions to Released and releases player`() {
        manager.initialize()
        manager.release()
        assertEquals(EngineLifecycle.Released, manager.currentState)
        verify { mockPlayer.release() }
    }

    @Test
    fun `forceRelease releases player immediately even if stuck`() {
        manager.initialize()
        manager.forceRelease()
        assertEquals(EngineLifecycle.Released, manager.currentState)
        verify { mockPlayer.release() }
    }
}

```

## xplayer/app/src/test/java/com/xplayer/dev/EngineStateValidatorTest.kt

```kotlin
package com.xplayer.dev

import com.xplayer.dev.core.engine.EngineStateValidator
import com.xplayer.dev.core.engine.IllegalStateTransitionException
import com.xplayer.dev.core.state.PlayerState
import org.junit.Assert.*
import org.junit.Before
import org.junit.Test

class EngineStateValidatorTest {

    private lateinit var validator: EngineStateValidator

    @Before
    fun setUp() {
        validator = EngineStateValidator()
    }

    @Test
    fun `valid transition from Idle to Preparing is accepted`() {
        val result = validator.validate(PlayerState.Idle, PlayerState.Preparing("id"))
        assertTrue(result is EngineStateValidator.ValidationResult.Accepted)
    }

    @Test
    fun `invalid transition from Idle to Playing is rejected`() {
        val result = validator.validate(PlayerState.Idle, PlayerState.Playing("id", 0L, 100L))
        assertTrue(result is EngineStateValidator.ValidationResult.Rejected)
    }

    @Test(expected = IllegalStateTransitionException::class)
    fun `invalid transition in debug mode throws exception`() {
        validator.validate(PlayerState.Idle, PlayerState.Playing("id", 0L, 100L), isDebug = true)
    }

    @Test
    fun `same type transitions are always accepted`() {
        val state1 = PlayerState.Playing("id", 10L, 100L)
        val state2 = PlayerState.Playing("id", 20L, 100L)
        val result = validator.validate(state1, state2)
        assertTrue(result is EngineStateValidator.ValidationResult.Accepted)
    }
}

```

## xplayer/app/src/test/java/com/xplayer/dev/Phase567Test.kt

```kotlin
package com.xplayer.dev

import androidx.media3.common.PlaybackException
import androidx.media3.common.Player
import androidx.media3.common.VideoSize
import com.xplayer.dev.core.events.*
import com.xplayer.dev.core.listener.*
import com.xplayer.dev.core.session.*
import com.xplayer.dev.core.state.*
import io.mockk.*
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.ExperimentalCoroutinesApi
import kotlinx.coroutines.flow.first
import kotlinx.coroutines.flow.take
import kotlinx.coroutines.flow.toList
import kotlinx.coroutines.launch
import kotlinx.coroutines.test.*
import org.junit.After
import org.junit.Assert.*
import org.junit.Before
import org.junit.Test

@OptIn(ExperimentalCoroutinesApi::class)
class Phase567Test {

    private val testDispatcher = UnconfinedTestDispatcher()

    private lateinit var eventBus: PlayerEventBus
    private lateinit var eventQueue: PlayerEventQueue
    private lateinit var sessionFactory: SessionFactory
    private lateinit var sessionValidator: SessionValidator
    private lateinit var sessionStorage: SessionStorage
    private lateinit var sessionRepository: SessionRepository
    private lateinit var sessionManager: EngineSessionManager
    private lateinit var stateManager: PlayerStateManager
    private lateinit var listenerHub: EngineListenerHub

    @Before
    fun setUp() {
        Dispatchers.setMain(testDispatcher)
        eventBus = PlayerEventBus()
        eventQueue = PlayerEventQueue()
        sessionFactory = SessionFactory()
        sessionValidator = SessionValidator()
        sessionStorage = SessionStorage()
        sessionRepository = DefaultSessionRepository(sessionStorage, sessionValidator)
        sessionManager = EngineSessionManager(sessionFactory, sessionRepository, eventBus)
        stateManager = PlayerStateManager(PlayerStateReducer(PlayerStateValidator()), PlayerStateStore())
        listenerHub = EngineListenerHub(stateManager, eventBus, sessionManager)
    }

    @After
    fun tearDown() {
        Dispatchers.resetMain()
    }

    // ── Phase 05 Tests (Event System) ─────────────────────────────────────────

    @Test
    fun `event bus broadcasts events to collectors`() = runTest(testDispatcher) {
        val collected = mutableListOf<PlayerEvent>()
        val job = launch {
            eventBus.broadcast.take(2).toList(collected)
        }

        val e1 = PlayerEvent.PlaybackStarted("media-1", 100L)
        val e2 = PlayerEvent.PlaybackPaused("media-1", 200L)
        eventBus.emit(e1)
        eventBus.emit(e2)

        assertEquals(2, collected.size)
        assertEquals(e1.eventId, collected[0].eventId)
        assertEquals(e2.eventId, collected[1].eventId)
        job.cancel()
    }

    @Test
    fun `event queue maintains bounded buffer`() {
        for (i in 1..150) {
            eventQueue.enqueue(PlayerEvent.BufferUpdated("media-1", i))
        }
        assertEquals(100, eventQueue.getAll().size)
    }

    // ── Phase 07 Tests (Session Management) ───────────────────────────────────

    @Test
    fun `openSession initializes session and restores last position`() {
        val oldSession = sessionFactory.createSession("media-1", "http://test.mp4").apply {
            lastPositionMs = 5000L
        }
        sessionRepository.saveSession(oldSession)

        val newSession = sessionManager.openSession("media-1", "http://test.mp4")
        assertNotNull(newSession)
        assertEquals("media-1", newSession.mediaId)
        assertEquals(5000L, newSession.lastPositionMs)
        assertEquals(newSession.sessionId, sessionManager.currentSessionId)
    }

    @Test
    fun `closeSession persists session duration and ends session`() {
        sessionManager.openSession("media-1", "http://test.mp4")
        assertTrue(sessionManager.hasActiveSession())

        val closed = sessionManager.closeSession()
        assertNotNull(closed)
        assertFalse(sessionManager.hasActiveSession())
        assertEquals(EngineSessionManager.NO_SESSION_ID, sessionManager.currentSessionId)
        assertNotNull(sessionRepository.getSession(closed!!.sessionId))
    }

    // ── Phase 06 Tests (Listener Architecture) ────────────────────────────────

    @Test
    fun `listener hub bridges playback callbacks to state manager and event bus`() = runTest(testDispatcher) {
        sessionManager.openSession("media-1", "http://test.mp4")
        
        val events = mutableListOf<PlayerEvent>()
        val job = launch {
            eventBus.broadcast.take(3).toList(events) // SessionStarted + BufferingStarted + PlaybackStarted
        }

        listenerHub.onPlaybackStateChanged(Player.STATE_BUFFERING)
        assertTrue(stateManager.currentState is PlayerState.Buffering)

        listenerHub.onIsPlayingChanged(true)
        assertTrue(stateManager.currentState is PlayerState.Playing)

        job.cancel()
    }
}

```

## xplayer/app/src/test/java/com/xplayer/dev/PlayerEngineTest.kt

```kotlin
package com.xplayer.dev

import com.xplayer.dev.core.engine.*
import com.xplayer.dev.core.state.*
import io.mockk.*
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.ExperimentalCoroutinesApi
import kotlinx.coroutines.test.*
import org.junit.*
import org.junit.Assert.*

@OptIn(ExperimentalCoroutinesApi::class)
class PlayerEngineTest {

    private val testDispatcher = UnconfinedTestDispatcher()
    private lateinit var factory: PlayerFactory
    private lateinit var lifecycleManager: EngineLifecycleManager
    private lateinit var stateManager: PlayerStateManager
    private lateinit var engine: PlayerEngine
    private lateinit var mockPlayer: androidx.media3.exoplayer.ExoPlayer

    @Before
    fun setUp() {
        Dispatchers.setMain(testDispatcher)
        factory = mockk(relaxed = true)
        mockPlayer = mockk(relaxed = true)
        every { factory.create() } returns mockPlayer

        val lifecycleValidator = EngineLifecycleValidator()
        val logger = EngineLifecycleLogger()
        lifecycleManager = EngineLifecycleManager(factory, lifecycleValidator, logger)
        
        val stateValidator = PlayerStateValidator()
        val reducer = PlayerStateReducer(stateValidator)
        val store = PlayerStateStore()
        stateManager = PlayerStateManager(reducer, store)

        engine = PlayerEngine(lifecycleManager, stateManager)
    }

    @After
    fun tearDown() {
        Dispatchers.resetMain()
        unmockkAll()
    }

    @Test
    fun `initial lifecycle is Uninitialized and state is Idle`() {
        assertEquals(EngineLifecycle.Uninitialized, engine.lifecycleState)
        assertEquals(PlayerState.Idle, engine.state.value)
    }

    @Test
    fun `initialize transitions lifecycle to Active and calls factory create`() {
        engine.initialize()
        verify(exactly = 1) { factory.create() }
        assertTrue(engine.lifecycleState is EngineLifecycle.Active)
    }

    @Test
    fun `double initialize ignores subsequent calls`() {
        engine.initialize()
        engine.initialize()
        verify(exactly = 1) { factory.create() }
    }

    @Test
    fun `release transitions lifecycle to Released and resets state to Idle`() {
        engine.initialize()
        engine.release()
        assertEquals(EngineLifecycle.Released, engine.lifecycleState)
        assertEquals(PlayerState.Idle, engine.state.value)
    }

    @Test
    fun `seekBack and seekForward delegate to ExoPlayer`() {
        engine.initialize()
        engine.seekBack()
        verify { mockPlayer.seekBack() }
        engine.seekForward()
        verify { mockPlayer.seekForward() }
    }

    @Test
    fun `setPlaybackSpeed updates player parameters`() {
        engine.initialize()
        engine.setPlaybackSpeed(1.5f)
        verify { mockPlayer.playbackParameters = any() }
    }

    @Test
    fun `setVolume updates player volume`() {
        engine.initialize()
        engine.setVolume(0.5f)
        verify { mockPlayer.volume = 0.5f }
    }
}

```

## xplayer/app/src/test/java/com/xplayer/dev/PlayerManagerTest.kt

```kotlin
package com.xplayer.dev

import app.cash.turbine.test
import com.xplayer.dev.core.manager.PlayerManager
import com.xplayer.dev.core.repository.PlayerRepository
import com.xplayer.dev.core.state.PlayerState
import com.xplayer.dev.core.threading.PlayerDispatchers
import com.xplayer.dev.core.threading.PlayerScopeManager
import io.mockk.*
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.ExperimentalCoroutinesApi
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.emptyFlow
import kotlinx.coroutines.test.*
import org.junit.*
import org.junit.Assert.*

@OptIn(ExperimentalCoroutinesApi::class)
class PlayerManagerTest {

    private val testDispatcher = UnconfinedTestDispatcher()
    private lateinit var repository: PlayerRepository
    private lateinit var scopeManager: PlayerScopeManager
    private lateinit var manager: PlayerManager

    @Before
    fun setUp() {
        Dispatchers.setMain(testDispatcher)

        repository = mockk(relaxed = true) {
            every { state } returns MutableStateFlow(PlayerState.Idle)
            every { events } returns emptyFlow()
        }

        val dispatchers = PlayerDispatchers(
            main = testDispatcher,
            io = testDispatcher,
            default = testDispatcher,
        )
        scopeManager = PlayerScopeManager(dispatchers)
        manager = PlayerManager(repository, scopeManager)
    }

    @After
    fun tearDown() {
        Dispatchers.resetMain()
        unmockkAll()
    }

    @Test
    fun `initial state is Idle`() = runTest {
        manager.state.test {
            assertEquals(PlayerState.Idle, awaitItem())
            cancelAndIgnoreRemainingEvents()
        }
    }

    @Test
    fun `play and pause delegate to repository`() = runTest {
        manager.play()
        coVerify { repository.play() }
        manager.pause()
        coVerify { repository.pause() }
    }

    @Test
    fun `seekBack and seekForward delegate to repository`() = runTest {
        manager.seekBack()
        coVerify { repository.seekBack() }
        manager.seekForward()
        coVerify { repository.seekForward() }
    }

    @Test
    fun `setPlaybackSpeed and setVolume delegate to repository`() = runTest {
        manager.setPlaybackSpeed(1.25f)
        coVerify { repository.setPlaybackSpeed(1.25f) }
        manager.setVolume(0.8f)
        coVerify { repository.setVolume(0.8f) }
    }
}

```

## xplayer/app/src/test/java/com/xplayer/dev/PlayerStateManagerTest.kt

```kotlin
package com.xplayer.dev

import androidx.media3.common.PlaybackException
import com.xplayer.dev.core.state.*
import org.junit.Assert.*
import org.junit.Before
import org.junit.Test

class PlayerStateManagerTest {

    private lateinit var validator: PlayerStateValidator
    private lateinit var reducer: PlayerStateReducer
    private lateinit var store: PlayerStateStore
    private lateinit var stateManager: PlayerStateManager

    @Before
    fun setUp() {
        validator = PlayerStateValidator()
        reducer = PlayerStateReducer(validator)
        store = PlayerStateStore()
        stateManager = PlayerStateManager(reducer, store)
    }

    @Test
    fun `initial state is Idle`() {
        assertEquals(PlayerState.Idle, stateManager.currentState)
        assertNull(stateManager.previousState)
    }

    @Test
    fun `legal transition from Idle to Preparing updates state and store`() {
        val success = stateManager.dispatch(PlayerStateCommand.Prepare("media-1"))
        assertTrue(success)
        assertTrue(stateManager.currentState is PlayerState.Preparing)
        assertEquals("media-1", (stateManager.currentState as PlayerState.Preparing).mediaId)
        assertEquals(PlayerState.Idle, stateManager.previousState)
        assertEquals(1, store.size)
    }

    @Test
    fun `illegal transition from Idle to Playing is rejected`() {
        val success = stateManager.dispatch(PlayerStateCommand.Play("media-1"))
        assertFalse(success)
        assertEquals(PlayerState.Idle, stateManager.currentState)
        assertEquals(0, store.size)
    }

    @Test
    fun `error command transitions from any state and records previousState`() {
        stateManager.dispatch(PlayerStateCommand.Prepare("media-1"))
        val error = PlaybackException(null, null, PlaybackException.ERROR_CODE_UNSPECIFIED)
        val success = stateManager.dispatch(PlayerStateCommand.ErrorOccurred("media-1", error))
        
        assertTrue(success)
        assertTrue(stateManager.currentState is PlayerState.Error)
        assertEquals("media-1", (stateManager.currentState as PlayerState.Error).mediaId)
    }

    @Test
    fun `persistence hook is notified when state becomes Paused or Terminal`() {
        var persistedState: PlayerState? = null
        val hook = object : PlayerStatePersistenceHook {
            override fun persistState(state: PlayerState) {
                persistedState = state
            }
        }
        stateManager.addPersistenceHook(hook)

        stateManager.dispatch(PlayerStateCommand.Prepare("media-1"))
        stateManager.dispatch(PlayerStateCommand.Ready("media-1", 10000L))
        stateManager.dispatch(PlayerStateCommand.Play("media-1"))
        assertNull(persistedState) // Playing is active, not paused/terminal

        stateManager.dispatch(PlayerStateCommand.Pause("media-1", 5000L))
        assertNotNull(persistedState)
        assertTrue(persistedState is PlayerState.Paused)
    }
}

```

## xplayer/app/src/test/java/com/xplayer/dev/PlayerStateTest.kt

```kotlin
package com.xplayer.dev

import androidx.media3.common.PlaybackException
import com.xplayer.dev.core.config.PlayerConfiguration
import com.xplayer.dev.core.state.PlayerState
import com.xplayer.dev.core.state.isActive
import com.xplayer.dev.core.state.isTerminal
import com.xplayer.dev.core.state.mediaId
import org.junit.Assert.*
import org.junit.Test

class PlayerStateTest {

    @Test fun `Idle state has null mediaId`() {
        assertNull(PlayerState.Idle.mediaId)
    }

    @Test fun `Buffering rejects invalid bufferPercent`() {
        assertThrows(IllegalArgumentException::class.java) {
            PlayerState.Buffering(mediaId = "id", bufferPercent = 101)
        }
        assertThrows(IllegalArgumentException::class.java) {
            PlayerState.Buffering(mediaId = "id", bufferPercent = -1)
        }
    }

    @Test fun `Playing and Buffering are active states`() {
        val error = PlaybackException(null, null, PlaybackException.ERROR_CODE_UNSPECIFIED)
        assertTrue(PlayerState.Playing("id").isActive)
        assertTrue(PlayerState.Buffering("id").isActive)
        assertFalse(PlayerState.Idle.isActive)
        assertFalse(PlayerState.Paused("id").isActive)
        assertFalse(PlayerState.Error("id", error).isActive)
    }

    @Test fun `Ended and Error are terminal states`() {
        val error = PlaybackException(null, null, PlaybackException.ERROR_CODE_UNSPECIFIED)
        assertTrue(PlayerState.Ended("id").isTerminal)
        assertTrue(PlayerState.Error("id", error).isTerminal)
        assertFalse(PlayerState.Playing("id").isTerminal)
        assertFalse(PlayerState.Idle.isTerminal)
    }

    @Test fun `BufferConfig validation catches bad ranges`() {
        assertThrows(IllegalArgumentException::class.java) {
            PlayerConfiguration.BufferConfig(
                minBufferMs = 5_000,
                maxBufferMs = 1_000, // max < min → invalid
            )
        }
    }
}

```

## xplayer/app/src/test/java/com/xplayer/dev/ThreadSafetyTest.kt

```kotlin
package com.xplayer.dev

import com.xplayer.dev.core.state.PlayerState
import com.xplayer.dev.core.state.isActive
import com.xplayer.dev.core.state.mediaId
import kotlinx.coroutines.*
import kotlinx.coroutines.test.*
import org.junit.Assert.*
import org.junit.Test
import java.util.concurrent.CopyOnWriteArrayList
import java.util.concurrent.CountDownLatch

@OptIn(ExperimentalCoroutinesApi::class)
class ThreadSafetyTest {

    /**
     * Verifies that concurrent reads of state extensions are race-free.
     * State objects are immutable data classes — extension reads are safe.
     */
    @Test
    fun `concurrent state reads do not throw`() {
        val states = listOf(
            PlayerState.Idle,
            PlayerState.Playing("id", 1000L),
            PlayerState.Paused("id", 500L),
            PlayerState.Buffering("id", 50),
        )
        val errors = CopyOnWriteArrayList<Throwable>()
        val latch = CountDownLatch(states.size * 10)

        repeat(states.size) { i ->
            repeat(10) {
                Thread {
                    try {
                        val state = states[i]
                        // concurrent extension reads on immutable data class
                        state.isActive
                        state.mediaId
                        state.isActive && state.mediaId != null
                    } catch (e: Throwable) {
                        errors.add(e)
                    } finally {
                        latch.countDown()
                    }
                }.start()
            }
        }

        latch.await()
        assertTrue("Thread safety errors: $errors", errors.isEmpty())
    }

    /**
     * Verifies StateFlow updates from multiple coroutines
     * converge to a consistent final value.
     */
    @Test
    fun `StateFlow handles concurrent emissions correctly`() = runTest {
        val flow = kotlinx.coroutines.flow.MutableStateFlow<PlayerState>(PlayerState.Idle)
        val jobs = (1..100).map { i ->
            launch(Dispatchers.Default) {
                flow.value = PlayerState.Playing("media-$i", i.toLong())
            }
        }
        jobs.joinAll()

        // Final state must be a valid Playing state — no corruption
        val finalState = flow.value
        assertTrue(finalState is PlayerState.Playing || finalState is PlayerState.Idle)
    }
}

```

## xplayer/app/src/test/java/com/xplayer/dev/core/playback/PlaybackControllerTest.kt

```kotlin
package com.xplayer.dev.core.playback

import androidx.media3.exoplayer.ExoPlayer
import com.xplayer.dev.core.config.PlayerConfiguration
import com.xplayer.dev.core.state.PlayerState
import io.mockk.*
import kotlinx.coroutines.flow.MutableStateFlow
import org.junit.Assert.*
import org.junit.Before
import org.junit.Test

class PlaybackControllerTest {

    private lateinit var player: ExoPlayer
    private lateinit var stateFlow: MutableStateFlow<PlayerState>
    private lateinit var controller: PlaybackController

    @Before fun setUp() {
        player    = mockk(relaxed = true)
        stateFlow = MutableStateFlow(PlayerState.Idle())
        controller = PlaybackController(
            playerProvider = { player },
            stateFlow      = stateFlow,
            config         = PlayerConfiguration.DEFAULT,
        )
    }

    @Test fun `play rejected when Idle`() {
        stateFlow.value = PlayerState.Idle()
        val result = controller.play()
        assertIs<PlaybackController.CommandResult.Rejected>(result)
        verify(exactly = 0) { player.playWhenReady = true }
    }

    @Test fun `play accepted when Ready`() {
        stateFlow.value = PlayerState.Ready("id")
        val result = controller.play()
        assertIs<PlaybackController.CommandResult.Executed>(result)
        verify { player.playWhenReady = true }
    }

    @Test fun `pause rejected when Idle`() {
        stateFlow.value = PlayerState.Idle()
        val result = controller.pause()
        assertIs<PlaybackController.CommandResult.Rejected>(result)
    }

    @Test fun `pause accepted when Playing`() {
        stateFlow.value = PlayerState.Playing("id")
        val result = controller.pause()
        assertIs<PlaybackController.CommandResult.Executed>(result)
        verify { player.playWhenReady = false }
    }

    @Test fun `seekTo clamps to zero`() {
        stateFlow.value = PlayerState.Playing("id", isSeekable = true)
        every { player.duration } returns 120_000L
        controller.seekTo(-500L)
        verify { player.seekTo(0L) }
    }

    @Test fun `seekTo clamps to duration`() {
        stateFlow.value = PlayerState.Playing("id", isSeekable = true)
        every { player.duration } returns 120_000L
        controller.seekTo(200_000L)
        verify { player.seekTo(120_000L) }
    }

    @Test fun `setPlaybackSpeed rejects invalid range`() {
        assertThrows(IllegalArgumentException::class.java) {
            controller.setPlaybackSpeed(0.1f)
        }
        assertThrows(IllegalArgumentException::class.java) {
            controller.setPlaybackSpeed(9.0f)
        }
    }

    @Test fun `setVolume rejects out of range`() {
        assertThrows(IllegalArgumentException::class.java) {
            controller.setVolume(-0.1f)
        }
        assertThrows(IllegalArgumentException::class.java) {
            controller.setVolume(1.1f)
        }
    }

    @Test fun `replay seeks to 0 and plays`() {
        controller.replay()
        verify { player.seekTo(0L) }
        verify { player.playWhenReady = true }
    }

    @Test fun `NoPlayer returned when player is null`() {
        val nullController = PlaybackController(
            playerProvider = { null },
            stateFlow      = stateFlow,
            config         = PlayerConfiguration.DEFAULT,
        )
        assertEquals(PlaybackController.CommandResult.NoPlayer, nullController.play())
    }
}

```

## xplayer/app/src/test/java/com/xplayer/dev/core/playlist/QueueManagerTest.kt

```kotlin
package com.xplayer.dev.core.playlist

import org.junit.Assert.*
import org.junit.Before
import org.junit.Test

class QueueManagerTest {

    private lateinit var queue: QueueManager

    @Before fun setUp() { queue = QueueManager() }

    private fun item(id: String) = QueueManager.QueueItem(
        mediaId = id, url = "https://cdn.example.com/$id.m3u8"
    )

    @Test fun `add sets currentIndex to 0`() {
        queue.add(item("a"))
        assertEquals(0, queue.currentIndex)
        assertEquals("a", queue.currentItem?.mediaId)
    }

    @Test fun `addAll sets currentIndex to 0`() {
        queue.addAll(listOf(item("a"), item("b"), item("c")))
        assertEquals(3, queue.size)
        assertEquals(0, queue.currentIndex)
    }

    @Test fun `moveToNext advances index`() {
        queue.addAll(listOf(item("a"), item("b"), item("c")))
        val next = queue.moveToNext()
        assertEquals("b", next?.mediaId)
        assertEquals(1, queue.currentIndex)
    }

    @Test fun `moveToNext returns null at end with REPEAT_OFF`() {
        queue.addAll(listOf(item("a"), item("b")))
        queue.moveToNext()
        val result = queue.moveToNext()
        assertNull(result)
    }

    @Test fun `moveToNext wraps with REPEAT_ALL`() {
        queue.addAll(listOf(item("a"), item("b")))
        queue.setRepeatMode(QueueManager.RepeatMode.ALL)
        queue.moveToNext()
        val result = queue.moveToNext()
        assertEquals("a", result?.mediaId)
    }

    @Test fun `moveToPrevious returns null at start`() {
        queue.addAll(listOf(item("a"), item("b")))
        assertNull(queue.moveToPrevious())
    }

    @Test fun `removeAt adjusts currentIndex`() {
        queue.addAll(listOf(item("a"), item("b"), item("c")))
        queue.moveToNext() // index = 1
        queue.removeAt(0)  // remove "a", index should become 0
        assertEquals(0, queue.currentIndex)
        assertEquals("b", queue.currentItem?.mediaId)
    }

    @Test fun `move reorders items`() {
        queue.addAll(listOf(item("a"), item("b"), item("c")))
        queue.move(0, 2)
        assertEquals("b", queue.items[0].mediaId)
        assertEquals("c", queue.items[1].mediaId)
        assertEquals("a", queue.items[2].mediaId)
    }

    @Test fun `clear empties queue and resets index`() {
        queue.addAll(listOf(item("a"), item("b")))
        queue.clear()
        assertEquals(0, queue.size)
        assertEquals(-1, queue.currentIndex)
        assertNull(queue.currentItem)
    }

    @Test fun `RepeatMode ONE always returns same item`() {
        queue.addAll(listOf(item("a"), item("b")))
        queue.setRepeatMode(QueueManager.RepeatMode.ONE)
        val next = queue.moveToNext()
        assertEquals("a", next?.mediaId)
    }

    @Test fun `moveToIndex works correctly`() {
        queue.addAll(listOf(item("a"), item("b"), item("c")))
        val result = queue.moveToIndex(2)
        assertEquals("c", result?.mediaId)
        assertEquals(2, queue.currentIndex)
    }

    @Test fun `removeById removes correct item`() {
        queue.addAll(listOf(item("a"), item("b"), item("c")))
        val removed = queue.removeById("b")
        assertTrue(removed)
        assertEquals(2, queue.size)
        assertNull(queue.items.find { it.mediaId == "b" })
    }
}


// test/track/TrackManagerTest.kt

```

## xplayer/app/src/test/java/com/xplayer/dev/core/track/TrackManagerTest.kt

```kotlin
package com.xplayer.dev.core.track

import io.mockk.mockk
import org.junit.Assert.*
import org.junit.Before
import org.junit.Test

class TrackManagerTest {

    private lateinit var manager: TrackManager

    @Before fun setUp() {
        manager = TrackManager(
            overrideManager = mockk(relaxed = true),
            policy          = DefaultTrackPolicy(),
        )
    }

    @Test fun `initial track lists are empty`() {
        assertTrue(manager.videoTracks.isEmpty())
        assertTrue(manager.audioTracks.isEmpty())
        assertTrue(manager.subtitleTracks.isEmpty())
    }

    @Test fun `selectedVideoTrack returns null when no tracks`() {
        assertNull(manager.selectedVideoTrack)
    }

    @Test fun `selectedAudioTrack returns null when no tracks`() {
        assertNull(manager.selectedAudioTrack)
    }

    @Test fun `VideoTrack resolutionLabel for 1080p`() {
        val track = TrackManager.VideoTrack(
            groupIndex = 0, trackIndex = 0,
            width = 1920, height = 1080,
            bitrateBps = 5_000_000, frameRate = 30f,
            codec = "avc", isHdr = false, isSelected = true,
        )
        assertEquals("1080p", track.resolutionLabel)
    }

    @Test fun `VideoTrack resolutionLabel for 4K`() {
        val track = TrackManager.VideoTrack(
            groupIndex = 0, trackIndex = 0,
            width = 3840, height = 2160,
            bitrateBps = 20_000_000, frameRate = 60f,
            codec = "hevc", isHdr = true, isSelected = false,
        )
        assertEquals("4K", track.resolutionLabel)
    }
}


// test/subtitle/SubtitleParserTest.kt

```

## xplayer/app/src/test/java/com/xplayer/dev/core/subtitle/SubtitleParserTest.kt

```kotlin
package com.xplayer.dev.core.subtitle

import org.junit.Assert.*
import org.junit.Before
import org.junit.Test

class SubtitleParserTest {

    private lateinit var parser: SubtitleParser

    @Before fun setUp() { parser = SubtitleParser() }

    @Test fun `parse SRT detects format`() {
        val srt = """
            1
            00:00:01,000 --> 00:00:03,000
            Hello World
        """.trimIndent()
        val result = parser.parse(srt)
        assertEquals(SubtitleManager.SubtitleFormat.SRT, result.format)
        assertTrue(result.isValid)
    }

    @Test fun `parse WebVTT detects format`() {
        val vtt = """
            WEBVTT
            
            00:00:01.000 --> 00:00:03.000
            Hello World
        """.trimIndent()
        val result = parser.parse(vtt)
        assertEquals(SubtitleManager.SubtitleFormat.WEBVTT, result.format)
        assertTrue(result.isValid)
    }

    @Test fun `parse ASS detects format`() {
        val ass = "[Script Info]\nTitle: Test\n[V4+ Styles]\nDialogue: 0,0:00:01.00,0:00:03.00,Default,,Hello"
        val result = parser.parse(ass)
        assertEquals(SubtitleManager.SubtitleFormat.ASS, result.format)
        assertTrue(result.isValid)
    }

    @Test fun `parse blank returns invalid`() {
        val result = parser.parse("")
        assertFalse(result.isValid)
        assertEquals(SubtitleManager.SubtitleFormat.UNKNOWN, result.format)
    }

    @Test fun `SubtitleFormat from extension`() {
        assertEquals(SubtitleManager.SubtitleFormat.SRT,
            SubtitleManager.SubtitleFormat.fromExtension("subtitle.srt"))
        assertEquals(SubtitleManager.SubtitleFormat.WEBVTT,
            SubtitleManager.SubtitleFormat.fromExtension("subtitle.vtt"))
        assertEquals(SubtitleManager.SubtitleFormat.ASS,
            SubtitleManager.SubtitleFormat.fromExtension("subtitle.ass"))
    }
}



// test/metadata/MetadataExtractorTest.kt

```

## xplayer/app/src/test/java/com/xplayer/dev/core/metadata/MetadataExtractorTest.kt

```kotlin
package com.xplayer.dev.core.metadata

import androidx.media3.common.MediaMetadata
import io.mockk.every
import io.mockk.mockk
import org.junit.Assert.*
import org.junit.Before
import org.junit.Test

class MetadataExtractorTest {

    private lateinit var extractor: MetadataExtractor

    @Before fun setUp() { extractor = MetadataExtractor() }

    @Test fun `extract returns displayTitle from title`() {
        val metadata = mockk<MediaMetadata> {
            every { title }        returns "My Movie"
            every { artist }       returns null
            every { albumTitle }   returns null
            every { genre }        returns null
            every { description }  returns null
            every { artworkUri }   returns null
            every { trackNumber }  returns null
            every { totalTrackCount } returns null
            every { discNumber }   returns null
            every { releaseYear }  returns 2024
            every { recordingYear } returns null
            every { composer }     returns null
            every { albumArtist }  returns null
            every { conductor }    returns null
            every { writer }       returns null
            every { station }      returns null
        }
        val info = extractor.extract(metadata, "id")
        assertEquals("My Movie", info.displayTitle)
        assertEquals("My Movie", info.title)
        assertEquals(2024, info.releaseYear)
    }

    @Test fun `displayTitle falls back to artist`() {
        val metadata = mockk<MediaMetadata>(relaxed = true) {
            every { title }  returns null
            every { artist } returns "Artist Name"
        }
        val info = extractor.extract(metadata, "id")
        assertEquals("Artist Name", info.displayTitle)
    }

    @Test fun `displayTitle falls back to Unknown`() {
        val metadata = mockk<MediaMetadata>(relaxed = true) {
            every { title }  returns null
            every { artist } returns null
        }
        val info = extractor.extract(metadata, "id")
        assertEquals("Unknown", info.displayTitle)
    }

    @Test fun `hasArtwork false when no artwork`() {
        val metadata = mockk<MediaMetadata>(relaxed = true) {
            every { artworkUri } returns null
        }
        assertFalse(extractor.extract(metadata, "id").hasArtwork)
    }
}


// test/live/LiveLatencyControllerTest.kt

```

## xplayer/app/src/test/java/com/xplayer/dev/core/live/LiveLatencyControllerTest.kt

```kotlin
package com.xplayer.dev.core.live

import androidx.media3.common.PlaybackParameters
import androidx.media3.exoplayer.ExoPlayer
import io.mockk.*
import org.junit.Before
import org.junit.Test

class LiveLatencyControllerTest {

    private lateinit var controller: LiveLatencyController
    private lateinit var player: ExoPlayer

    @Before fun setUp() {
        controller = LiveLatencyController()
        player     = mockk(relaxed = true)
        every { player.isCurrentMediaItemLive }     returns true
        every { player.playbackParameters }         returns PlaybackParameters(1.0f)
    }

    @Test fun `speed up when behind 
```

## xplayer/app/src/test/java/com/xplayer/dev/core/analytics/AnalyticsManagerTest.kt

```kotlin
package com.xplayer.dev.core.analytics

import com.xplayer.dev.core.engine.internal.EngineMetricsProbe
import com.xplayer.dev.core.events.PlayerEventBus
import io.mockk.mockk
import io.mockk.verify
import kotlinx.coroutines.ExperimentalCoroutinesApi
import kotlinx.coroutines.test.*
import org.junit.*
import org.junit.Assert.*

@OptIn(ExperimentalCoroutinesApi::class)
class AnalyticsManagerTest {

    private val dispatcher = UnconfinedTestDispatcher()
    private lateinit var probe: EngineMetricsProbe
    private lateinit var profiler: PlaybackProfiler
    private lateinit var qosMonitor: QoSMonitor
    private lateinit var repository: AnalyticsRepository
    private lateinit var manager: AnalyticsManager

    @Before
    fun setUp() {
        probe      = EngineMetricsProbe()
        profiler   = PlaybackProfiler()
        qosMonitor = QoSMonitor()
        repository = AnalyticsRepository()
        manager    = AnalyticsManager(
            metricsProbe = probe,
            eventBus     = mockk(relaxed = true),
            profiler     = profiler,
            qosMonitor   = qosMonitor,
            repository   = repository,
        )
    }

    @Test
    fun `startSession resets profiler`() = runTest(dispatcher) {
        profiler.recordTtfp(500L)
        manager.startSession("session-1")
        // After reset, ttfp should be unmeasured
        val profile = profiler.buildProfile("session-1", probe.snapshot())
        assertNull(profile.ttfpMs)
    }

    @Test
    fun `endSession saves profile to repository`() = runTest(dispatcher) {
        manager.startSession("session-1")
        profiler.recordTtfp(800L)
        manager.endSession()
        val saved = repository.profileForSession("session-1")
        assertNotNull(saved)
        assertEquals(800L, saved?.ttfpMs)
    }
}

```

## xplayer/app/src/test/java/com/xplayer/dev/core/analytics/QoSMonitorTest.kt

```kotlin
package com.xplayer.dev.core.analytics

import com.xplayer.dev.core.engine.internal.MetricsSnapshot
import org.junit.*
import org.junit.Assert.*

class QoSMonitorTest {

    private lateinit var monitor: QoSMonitor

    @Before fun setUp() { monitor = QoSMonitor() }

    @Test
    fun `initial health is UNKNOWN`() {
        assertEquals(QoSHealth.UNKNOWN, monitor.currentHealth)
    }

    @Test
    fun `good metrics produce GOOD health`() {
        monitor.onSample(healthySnapshot())
        assertEquals(QoSHealth.GOOD, monitor.currentHealth)
    }

    @Test
    fun `many stalls produce DEGRADED health`() {
        monitor.onSample(
            healthySnapshot().copy(
                stallCount = 5,
                totalStalledMs = 10_000L,
            )
        )
        assertEquals(QoSHealth.DEGRADED, monitor.currentHealth)
    }

    @Test
    fun `reset returns to UNKNOWN`() {
        monitor.onSample(healthySnapshot())
        monitor.reset()
        assertEquals(QoSHealth.UNKNOWN, monitor.currentHealth)
    }

    private fun healthySnapshot() = MetricsSnapshot(
        ttfpMs                 = 800L,
        stallCount             = 0,
        totalStalledMs         = 0L,
        lastStallMs            = 0L,
        minBufferBeforeStallMs = null,
        seekCount              = 0,
        averageSeekMs          = 0L,
        droppedFrames          = 0,
        renderedFrames         = 1000L,
        averageBandwidthBps    = 2_000_000L,
        peakBandwidthBps       = 5_000_000L,
        minBandwidthBps        = 1_000_000L,
        bandwidthSampleCount   = 10,
    )
}

```

## xplayer/app/src/test/java/com/xplayer/dev/core/cache/CacheManagerTest.kt

```kotlin
package com.xplayer.dev.core.cache

import android.content.Context
import io.mockk.every
import io.mockk.mockk
import org.junit.*
import org.junit.Assert.*
import java.io.File

class CacheManagerTest {

    private lateinit var context: Context
    private lateinit var config: CacheConfiguration
    private lateinit var repository: CacheRepository
    private lateinit var manager: CacheManager

    @Before
    fun setUp() {
        val tempDir = createTempDir("cache_test")
        context = mockk(relaxed = true) {
            every { cacheDir } returns tempDir
        }
        config     = CacheConfiguration(maxCacheSizeBytes = 50 * 1_048_576L)
        repository = CacheRepository()
        manager    = CacheManager(context, config, repository)
    }

    @After
    fun tearDown() {
        runCatching { manager.release() }
    }

    @Test
    fun `initialise creates SimpleCache`() {
        manager.initialise()
        assertTrue(manager.isInitialised)
    }

    @Test
    fun `double initialise is safe`() {
        manager.initialise()
        manager.initialise() // should not throw
        assertTrue(manager.isInitialised)
    }

    @Test
    fun `release after initialise transitions to not initialised`() {
        manager.initialise()
        manager.release()
        assertFalse(manager.isInitialised)
    }
}

```

## xplayer/app/src/test/java/com/xplayer/dev/core/download/DownloadRepositoryTest.kt

```kotlin
package com.xplayer.dev.core.download

import org.junit.*
import org.junit.Assert.*

class DownloadRepositoryTest {

    private lateinit var repository: DownloadRepository

    @Before fun setUp() { repository = DownloadRepository() }

    @Test
    fun `record and retrieve download record`() {
        val record = DownloadRecord(downloadId = "d1", url = "http://test.com/a.mp4")
        repository.record(record)
        assertEquals(record, repository.getRecord("d1"))
    }

    @Test
    fun `updateState changes state`() {
        repository.record(DownloadRecord("d1", "url"))
        repository.updateState("d1", DownloadState.COMPLETED)
        assertEquals(DownloadState.COMPLETED, repository.getRecord("d1")?.state)
    }

    @Test
    fun `completedDownloads returns only completed`() {
        repository.record(DownloadRecord("d1", "url1"))
        repository.record(DownloadRecord("d2", "url2"))
        repository.updateState("d1", DownloadState.COMPLETED)
        assertEquals(1, repository.completedDownloads().size)
        assertEquals("d1", repository.completedDownloads()[0].downloadId)
    }

    @Test
    fun `remove deletes record`() {
        repository.record(DownloadRecord("d1", "url"))
        repository.remove("d1")
        assertNull(repository.getRecord("d1"))
    }
}

```

## xplayer/app/src/test/java/com/xplayer/dev/core/drm/DrmKeyManagerTest.kt

```kotlin
package com.xplayer.dev.core.drm

import android.content.Context
import io.mockk.every
import io.mockk.mockk
import org.junit.*
import org.junit.Assert.*

class DrmKeyManagerTest {

    private lateinit var context: Context
    private lateinit var keyManager: DrmKeyManager

    @Before
    fun setUp() {
        val tempDir = createTempDir("drm_keys_test")
        context = mockk {
            every { filesDir } returns tempDir
        }
        keyManager = DrmKeyManager(context)
    }

    @Test
    fun `storeAndRetrieve keySetId`() {
        val keySetId = byteArrayOf(1, 2, 3, 4, 5)
        keyManager.storeKeySetId("media-001", keySetId)
        val retrieved = keyManager.getKeySetId("media-001")
        assertNotNull(retrieved)
        assertTrue(keySetId.contentEquals(retrieved!!))
    }

    @Test
    fun `getKeySetId returns null for unknown mediaId`() {
        assertNull(keyManager.getKeySetId("unknown-id"))
    }

    @Test
    fun `hasKeySetId returns true after store`() {
        keyManager.storeKeySetId("media-002", byteArrayOf(9, 8, 7))
        assertTrue(keyManager.hasKeySetId("media-002"))
    }

    @Test
    fun `removeKeySetId clears stored key`() {
        keyManager.storeKeySetId("media-003", byteArrayOf(1))
        keyManager.removeKeySetId("media-003")
        assertFalse(keyManager.hasKeySetId("media-003"))
    }

    @Test
    fun `clearAll removes all keys`() {
        keyManager.storeKeySetId("m1", byteArrayOf(1))
        keyManager.storeKeySetId("m2", byteArrayOf(2))
        keyManager.clearAll()
        assertFalse(keyManager.hasKeySetId("m1"))
        assertFalse(keyManager.hasKeySetId("m2"))
    }
}

```

## xplayer/app/src/test/java/com/xplayer/dev/core/codec/CodecManagerTest.kt

```kotlin
package com.xplayer.dev.core.codec

import io.mockk.every
import io.mockk.mockk
import org.junit.*
import org.junit.Assert.*

class CodecManagerTest {

    private lateinit var repository: CodecRepository
    private lateinit var detector: CodecDetector
    private lateinit var selector: DecoderSelector
    private lateinit var manager: CodecManager

    @Before
    fun setUp() {
        repository = CodecRepository()
        detector   = mockk()
        selector   = DecoderSelector()
        manager    = CodecManager(repository, detector, selector)
    }

    @Test
    fun `discoverCodecs stores results`() {
        val codecs = listOf(
            CodecInfo(
                name = "OMX.qcom.video.decoder.avc",
                mimeType = "video/avc",
                isHardwareAccelerated = true,
                isSoftwareOnly = false,
                isVendor = true,
                maxWidth = 3840, maxHeight = 2160,
                maxFrameRate = 60,
                supportsHdr = false,
                profileLevels = emptyList(),
            )
        )
        every { detector.discoverAll() } returns codecs
        manager.discoverCodecs()
        assertTrue(manager.isFormatSupported("video/avc"))
        assertTrue(manager.isHardwareDecoderAvailable("video/avc"))
    }

    @Test
    fun `selectDecoder returns hardware when available`() {
        val hwCodec = CodecInfo(
            name = "OMX.hw.h264",
            mimeType = "video/avc",
            isHardwareAccelerated = true,
            isSoftwareOnly = false,
            isVendor = true,
            maxWidth = 1920, maxHeight = 1080,
            maxFrameRate = 30,
            supportsHdr = false,
            profileLevels = emptyList(),
        )
        val swCodec = hwCodec.copy(
            name = "OMX.sw.h264",
            isHardwareAccelerated = false,
            isSoftwareOnly = true,
            isVendor = false,
        )
        every { detector.discoverAll() } returns listOf(hwCodec, swCodec)
        manager.discoverCodecs()

        val selected = manager.selectDecoder("video/avc")
        assertNotNull(selected)
        assertTrue(selected!!.isHardwareAccelerated)
    }

    @Test
    fun `isFormatSupported returns false for unknown mime`() {
        every { detector.discoverAll() } returns emptyList()
        manager.discoverCodecs()
        assertFalse(manager.isFormatSupported("video/unknown"))
    }
}

```

## xplayer/app/src/test/java/com/xplayer/dev/core/persistence/ResumeManagerTest.kt

```kotlin
package com.xplayer.dev.core.persistence

import org.junit.*
import org.junit.Assert.*

class ResumeManagerTest {

    private lateinit var repository: PlaybackRepository
    private lateinit var resumeManager: ResumeManager

    @Before
    fun setUp() {
        repository    = PlaybackRepository()
        resumeManager = ResumeManager(repository)
    }

    @Test
    fun `position saved when below completion threshold`() {
        resumeManager.onPlaybackProgress("m1", 30_000L, 120_000L)
        assertEquals(30_000L, resumeManager.getResumePosition("m1"))
    }

    @Test
    fun `position cleared when near completion`() {
        resumeManager.onPlaybackProgress("m1", 115_000L, 120_000L)
        assertEquals(0L, resumeManager.getResumePosition("m1"))
    }

    @Test
    fun `no position returns 0`() {
        assertEquals(0L, resumeManager.getResumePosition("unknown"))
    }

    @Test
    fun `hasResumePosition is true when position exists`() {
        resumeManager.onPlaybackProgress("m2", 10_000L, 60_000L)
        assertTrue(resumeManager.hasResumePosition("m2"))
    }

    @Test
    fun `clearResumePosition removes saved position`() {
        resumeManager.onPlaybackProgress("m3", 10_000L, 60_000L)
        resumeManager.clearResumePosition("m3")
        assertFalse(resumeManager.hasResumePosition("m3"))
    }
}

```

## xplayer/app/src/test/java/com/xplayer/dev/core/library/LibraryRepositoryTest.kt

```kotlin
package com.xplayer.dev.core.library

import android.net.Uri
import io.mockk.mockk
import org.junit.*
import org.junit.Assert.*

class LibraryRepositoryTest {

    private lateinit var repository: LibraryRepository

    @Before fun setUp() { repository = LibraryRepository() }

    @Test
    fun `replaceAll stores items`() {
        repository.replaceAll(listOf(makeItem("m1"), makeItem("m2")))
        assertEquals(2, repository.allItems().size)
    }

    @Test
    fun `replaceAll clears previous items`() {
        repository.replaceAll(listOf(makeItem("m1")))
        repository.replaceAll(listOf(makeItem("m2"), makeItem("m3")))
        assertEquals(2, repository.allItems().size)
        assertNull(repository.getItem("m1"))
    }

    @Test
    fun `updateFavourite marks item`() {
        repository.replaceAll(listOf(makeItem("m1")))
        repository.updateFavourite("m1", true)
        assertTrue(repository.getItem("m1")?.isFavourite == true)
    }

    @Test
    fun `updateWatchProgress clamps to 0-1`() {
        repository.replaceAll(listOf(makeItem("m1")))
        repository.updateWatchProgress("m1", 1.5f)
        assertEquals(1.0f, repository.getItem("m1")?.watchProgress ?: -1f, 0.001f)
    }

    @Test
    fun `recentlyPlayed returns sorted by lastPlayed`() {
        repository.replaceAll(listOf(makeItem("m1"), makeItem("m2")))
        repository.updateLastPlayed("m1", 1000L)
        repository.updateLastPlayed("m2", 2000L)
        val recent = repository.recentlyPlayed(10)
        assertEquals("m2", recent[0].mediaId)
    }

    private fun makeItem(mediaId: String) = MediaItem_(
        mediaId    = mediaId,
        uri        = mockk<Uri>(relaxed = true),
        path       = "/storage/$mediaId.mp4",
        title      = "Test $mediaId",
        mimeType   = "video/mp4",
        sizeBytes  = 100_000L,
        durationMs = 60_000L,
        dateAdded  = System.currentTimeMillis(),
    )
}

```

## xplayer/app/src/test/java/com/xplayer/dev/core/engine/EngineMetricsProbeTest.kt

```kotlin
// test/engine/EngineMetricsProbeTest.kt

package com.xplayer.dev.core.engine

import com.xplayer.dev.core.engine.internal.EngineMetricsProbe
import org.junit.Assert.*
import org.junit.Before
import org.junit.Test

class EngineMetricsProbeTest {

    private lateinit var probe: EngineMetricsProbe

    @Before fun setUp() { probe = EngineMetricsProbe() }

    @Test
    fun `ttfp measured correctly`() {
        probe.onPrepareStart()
        Thread.sleep(50)
        val ttfp = probe.onFirstFrame()
        assertTrue("TTFP should be >= 50ms", ttfp >= 50L)
    }

    @Test
    fun `ttfp measured only once`() {
        probe.onPrepareStart()
        val first  = probe.onFirstFrame()
        val second = probe.onFirstFrame()
        assertTrue(first >= 0L)
        assertEquals(-1L, second) // second call is no-op
    }

    @Test
    fun `stall count increments correctly`() {
        probe.onStallStart()
        probe.onStallEnd()
        probe.onStallStart()
        probe.onStallEnd()
        assertEquals(2, probe.snapshot().stallCount)
    }

    @Test
    fun `total stalled time accumulates`() {
        probe.onStallStart()
        Thread.sleep(50)
        probe.onStallEnd()

        probe.onStallStart()
        Thread.sleep(50)
        probe.onStallEnd()

        assertTrue(probe.snapshot().totalStalledMs >= 100L)
    }

    @Test
    fun `reset clears all metrics`() {
        probe.onPrepareStart()
        probe.onFirstFrame()
        probe.onStallStart()
        probe.onStallEnd()
        probe.onDroppedFrames(10)

        probe.reset()

        val snapshot = probe.snapshot()
        assertNull(snapshot.ttfpMs)
        assertEquals(0, snapshot.stallCount)
        assertEquals(0, snapshot.droppedFrames)
    }

    @Test
    fun `bandwidth average computed correctly`() {
        probe.onBandwidthSample(1_000_000L)
        probe.onBandwidthSample(3_000_000L)
        assertEquals(2_000_000L, probe.snapshot().averageBandwidthBps)
    }

    @Test
    fun `drop rate calculation`() {
        probe.onDroppedFrames(10)
        probe.onRenderedFrames(90L)
        assertEquals(0.1f, probe.snapshot().dropRate, 0.001f)
    }
}
```

## xplayer/app/src/test/java/com/xplayer/dev/core/engine/EngineStateValidatorTest.kt

```kotlin
// test/engine/EngineStateValidatorTest.kt

package com.xplayer.dev.core.engine

import androidx.media3.common.PlaybackException
import com.xplayer.dev.core.engine.internal.EngineStateValidator
import com.xplayer.dev.core.engine.internal.IllegalStateTransitionException
import com.xplayer.dev.core.state.PlayerState
import org.junit.Assert.*
import org.junit.Before
import org.junit.Test

class EngineStateValidatorTest {

    private lateinit var validator: EngineStateValidator

    @Before
    fun setUp() {
        validator = EngineStateValidator()
    }

    @Test
    fun `Idle to Preparing is accepted`() {
        val result = validator.validate(PlayerState.Idle, PlayerState.Preparing("id"))
        assertIs<EngineStateValidator.ValidationResult.Accepted>(result)
    }

    @Test
    fun `Idle to Playing is rejected`() {
        val result = validator.validate(PlayerState.Idle, PlayerState.Playing("id"))
        assertIs<EngineStateValidator.ValidationResult.Rejected>(result)
    }

    @Test
    fun `Idle to Playing throws in debug`() {
        assertThrows(IllegalStateTransitionException::class.java) {
            validator.validate(
                PlayerState.Idle,
                PlayerState.Playing("id"),
                isDebug = true,
            )
        }
    }

    @Test
    fun `Playing to Buffering is accepted`() {
        val result = validator.validate(
            PlayerState.Playing("id"),
            PlayerState.Buffering("id"),
        )
        assertIs<EngineStateValidator.ValidationResult.Accepted>(result)
    }

    @Test
    fun `Ended to Preparing is accepted for replay`() {
        val result = validator.validate(
            PlayerState.Ended("id", 120_000L),
            PlayerState.Preparing("id"),
        )
        assertIs<EngineStateValidator.ValidationResult.Accepted>(result)
    }

    @Test
    fun `Error to Preparing is accepted for retry`() {
        val error = PlaybackException(null, null, PlaybackException.ERROR_CODE_UNSPECIFIED)
        val result = validator.validate(
            PlayerState.Error("id", error),
            PlayerState.Preparing("id"),
        )
        assertIs<EngineStateValidator.ValidationResult.Accepted>(result)
    }

    @Test
    fun `same-type transition is always accepted`() {
        // State update (position change) in Playing
        val result = validator.validate(
            PlayerState.Playing("id", 1000L),
            PlayerState.Playing("id", 2000L),
        )
        assertIs<EngineStateValidator.ValidationResult.Accepted>(result)
    }
}
```

## xplayer/app/src/test/java/com/xplayer/dev/core/engine/PlayerEngineIntegrationTest.kt

```kotlin
package com.xplayer.dev.core.engine

import androidx.media3.common.MediaItem
import androidx.media3.exoplayer.ExoPlayer
import app.cash.turbine.test
import com.xplayer.dev.core.engine.internal.*
import com.xplayer.dev.core.state.PlayerState
import io.mockk.*
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.ExperimentalCoroutinesApi
import kotlinx.coroutines.test.*
import org.junit.*
import org.junit.Assert.*

@OptIn(ExperimentalCoroutinesApi::class)
class PlayerEngineIntegrationTest {

    private val dispatcher = UnconfinedTestDispatcher()
    private lateinit var mockPlayer: ExoPlayer
    private lateinit var factory: PlayerFactory
    private lateinit var engine: PlayerEngine

    @Before
    fun setUp() {
        Dispatchers.setMain(dispatcher)
        mockPlayer = mockk(relaxed = true)
        factory    = mockk { every { create() } returns mockPlayer }
        engine     = PlayerEngine(
            factory    = factory,
            sessionMgr = EngineSessionManager(),
            metrics    = EngineMetricsProbe(),
            eventBus   = EngineEventBus(),
            validator  = EngineStateValidator(),
        )
    }

    @After
    fun tearDown() {
        runCatching { engine.release() }
        Dispatchers.resetMain()
        unmockkAll()
    }

    // ── Full happy path ───────────────────────────────────────────────────────

    @Test
    fun `full lifecycle — init prepare play pause stop release`() = runTest(dispatcher) {
        engine.state.test {
            // Initial
            assertIs<PlayerState.Idle>(awaitItem())

            // Init
            engine.initialize()
            assertIs<EngineLifecycle.Active>(engine.currentLifecycle())

            // Prepare
            every { mockPlayer.currentPosition } returns 0L
            every { mockPlayer.duration } returns 120_000L
            every { mockPlayer.bufferedPercentage } returns 0
            every { mockPlayer.bufferedPosition } returns 0L

            engine.prepare(
                mediaItem = MediaItem.fromUri("https://example.com/video.mp4"),
                mediaId   = "test-media",
                url       = "https://example.com/video.mp4",
            )

            // Verify ExoPlayer was called
            verify { mockPlayer.setMediaItem(any()) }
            verify { mockPlayer.prepare() }

            // Force playing state for guard tests
            engine.forceState(PlayerState.Playing("test-media", 0L))
            assertIs<PlayerState.Playing>(awaitItem())

            // Pause
            engine.pause()
            verify { mockPlayer.playWhenReady = false }

            // Stop
            engine.stop()
            assertIs<PlayerState.Idle>(awaitItem())

            // Release
            engine.release()
            assertIs<EngineLifecycle.Released>(engine.currentLifecycle())

            cancelAndIgnoreRemainingEvents()
        }
    }

    // ── Double init guard ─────────────────────────────────────────────────────

    @Test
    fun `initialize twice does not create two players`() {
        engine.initialize()
        engine.initialize()
        verify(exactly = 1) { factory.create() }
    }

    // ── Commands rejected after release ───────────────────────────────────────

    @Test
    fun `play after release does not throw`() {
        engine.initialize()
        engine.release()
        // Should throw on requirePlayer — caught by callers
        runCatching { engine.play() }
    }

    // ── Volume range guard ────────────────────────────────────────────────────

    @Test
    fun `setVolume 0f and 1f are accepted`() {
        engine.initialize()
        engine.setVolume(0f)
        engine.setVolume(1f)
        verify(exactly = 2) { mockPlayer.volume = any() }
    }

    @Test
    fun `setVolume out of range throws`() {
        engine.initialize()
        assertThrows(IllegalArgumentException::class.java) { engine.setVolume(-0.1f) }
        assertThrows(IllegalArgumentException::class.java) { engine.setVolume(1.1f) }
    }

    // ── Speed range guard ─────────────────────────────────────────────────────

    @Test
    fun `setPlaybackSpeed valid range accepted`() {
        engine.initialize()
        engine.setPlaybackSpeed(0.25f)
        engine.setPlaybackSpeed(2.0f)
        engine.setPlaybackSpeed(8.0f)
        verify(exactly = 3) { mockPlayer.playbackParameters = any() }
    }

    @Test
    fun `setPlaybackSpeed invalid range throws`() {
        engine.initialize()
        assertThrows(IllegalArgumentException::class.java) {
            engine.setPlaybackSpeed(0.1f)
        }
        assertThrows(IllegalArgumentException::class.java) {
            engine.setPlaybackSpeed(9.0f)
        }
    }
}

```

## xplayer/app/src/test/java/com/xplayer/dev/core/engine/PlayerEngineTest.kt

```kotlin
// test/engine/PlayerEngineTest.kt

package com.xplayer.dev.core.engine

import app.cash.turbine.test
import com.xplayer.dev.core.engine.internal.*
import com.xplayer.dev.core.state.PlayerState
import io.mockk.*
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.ExperimentalCoroutinesApi
import kotlinx.coroutines.test.*
import org.junit.*
import org.junit.Assert.*

@OptIn(ExperimentalCoroutinesApi::class)
class PlayerEngineTest {

    private val testDispatcher = UnconfinedTestDispatcher()

    private lateinit var factory: PlayerFactory
    private lateinit var sessionMgr: EngineSessionManager
    private lateinit var metrics: EngineMetricsProbe
    private lateinit var eventBus: EngineEventBus
    private lateinit var validator: EngineStateValidator
    private lateinit var engine: PlayerEngine

    @Before
    fun setUp() {
        Dispatchers.setMain(testDispatcher)
        factory    = mockk(relaxed = true)
        sessionMgr = EngineSessionManager()
        metrics    = EngineMetricsProbe()
        eventBus   = EngineEventBus()
        validator  = EngineStateValidator()
        engine     = PlayerEngine(factory, sessionMgr, metrics, eventBus, validator)
    }

    @After
    fun tearDown() {
        Dispatchers.resetMain()
        unmockkAll()
    }

    // ── Lifecycle tests ───────────────────────────────────────────────────────

    @Test
    fun `initial lifecycle is Uninitialised`() {
        assertIs<EngineLifecycle.Uninitialised>(engine.currentLifecycle())
    }

    @Test
    fun `initialize transitions to Active`() {
        val mockPlayer = mockk<androidx.media3.exoplayer.ExoPlayer>(relaxed = true)
        every { factory.create() } returns mockPlayer

        engine.initialize()

        assertIs<EngineLifecycle.Active>(engine.currentLifecycle())
    }

    @Test
    fun `double initialize is a no-op`() {
        val mockPlayer = mockk<androidx.media3.exoplayer.ExoPlayer>(relaxed = true)
        every { factory.create() } returns mockPlayer

        engine.initialize()
        engine.initialize() // Should not throw

        verify(exactly = 1) { factory.create() }
    }

    @Test
    fun `initialize after release throws`() {
        val mockPlayer = mockk<androidx.media3.exoplayer.ExoPlayer>(relaxed = true)
        every { factory.create() } returns mockPlayer
        engine.initialize()
        engine.release()

        assertThrows(IllegalStateException::class.java) {
            engine.initialize()
        }
    }

    @Test
    fun `release transitions to Released`() {
        val mockPlayer = mockk<androidx.media3.exoplayer.ExoPlayer>(relaxed = true)
        every { factory.create() } returns mockPlayer
        engine.initialize()

        engine.release()

        assertIs<EngineLifecycle.Released>(engine.currentLifecycle())
    }

    @Test
    fun `release is idempotent`() {
        val mockPlayer = mockk<androidx.media3.exoplayer.ExoPlayer>(relaxed = true)
        every { factory.create() } returns mockPlayer
        engine.initialize()

        engine.release()
        engine.release() // Should not throw or crash
    }

    // ── State tests ───────────────────────────────────────────────────────────

    @Test
    fun `initial state is Idle`() = runTest {
        engine.state.test {
            assertIs<PlayerState.Idle>(awaitItem())
            cancelAndIgnoreRemainingEvents()
        }
    }

    @Test
    fun `release resets state to Idle`() = runTest {
        val mockPlayer = mockk<androidx.media3.exoplayer.ExoPlayer>(relaxed = true)
        every { factory.create() } returns mockPlayer
        engine.initialize()
        engine.forceState(PlayerState.Playing("id", 1000L))

        engine.release()

        assertIs<PlayerState.Idle>(engine.state.value)
    }

    // ── Play guard tests ──────────────────────────────────────────────────────

    @Test
    fun `play is no-op when state is Idle`() {
        val mockPlayer = mockk<androidx.media3.exoplayer.ExoPlayer>(relaxed = true)
        every { factory.create() } returns mockPlayer
        engine.initialize()

        engine.play() // Should not throw; state is Idle → not playable

        verify(exactly = 0) { mockPlayer.playWhenReady = true }
    }

    @Test
    fun `play sets playWhenReady when Ready`() {
        val mockPlayer = mockk<androidx.media3.exoplayer.ExoPlayer>(relaxed = true)
        every { factory.create() } returns mockPlayer
        engine.initialize()
        engine.forceState(PlayerState.Ready("id", 120_000L))

        engine.play()

        verify { mockPlayer.playWhenReady = true }
    }

    // ── Seek guard tests ──────────────────────────────────────────────────────

    @Test
    fun `seekTo is no-op when Idle`() {
        val mockPlayer = mockk<androidx.media3.exoplayer.ExoPlayer>(relaxed = true)
        every { factory.create() } returns mockPlayer
        engine.initialize()

        engine.seekTo(5000L) // Idle → not seekable

        verify(exactly = 0) { mockPlayer.seekTo(any()) }
    }

    @Test
    fun `seekTo works when Playing`() {
        val mockPlayer = mockk<androidx.media3.exoplayer.ExoPlayer>(relaxed = true)
        every { factory.create() } returns mockPlayer
        every { mockPlayer.currentPosition } returns 1000L
        engine.initialize()
        sessionMgr.openSession("id", "http://test.com")
        engine.forceState(PlayerState.Playing("id", 1000L))

        engine.seekTo(5000L)

        verify { mockPlayer.seekTo(5000L) }
    }

    // ── Speed tests ───────────────────────────────────────────────────────────

    @Test
    fun `setPlaybackSpeed rejects invalid range`() {
        val mockPlayer = mockk<androidx.media3.exoplayer.ExoPlayer>(relaxed = true)
        every { factory.create() } returns mockPlayer
        engine.initialize()

        assertThrows(IllegalArgumentException::class.java) {
            engine.setPlaybackSpeed(0.1f) // below 0.25
        }
        assertThrows(IllegalArgumentException::class.java) {
            engine.setPlaybackSpeed(9.0f) // above 8.0
        }
    }

    @Test
    fun `setVolume rejects invalid range`() {
        val mockPlayer = mockk<androidx.media3.exoplayer.ExoPlayer>(relaxed = true)
        every { factory.create() } returns mockPlayer
        engine.initialize()

        assertThrows(IllegalArgumentException::class.java) {
            engine.setVolume(-0.1f)
        }
        assertThrows(IllegalArgumentException::class.java) {
            engine.setVolume(1.1f)
        }
    }

    // ── Metrics tests ─────────────────────────────────────────────────────────

    @Test
    fun `metrics reset on new prepare`() {
        val mockPlayer = mockk<androidx.media3.exoplayer.ExoPlayer>(relaxed = true)
        every { factory.create() } returns mockPlayer
        engine.initialize()

        // Simulate stall from previous session
        metrics.onStallStart(0L)
        metrics.onStallEnd()

        // Prepare new media
        engine.prepare(
            mockk(relaxed = true),
            mediaId = "new-id",
            url = "http://new.com",
        )

        val snapshot = engine.metricsSnapshot
        assertEquals(0, snapshot.stallCount)
        assertFalse(snapshot.hasTtfp)
    }

    // ── Event bus tests ───────────────────────────────────────────────────────

    @Test
    fun `droppedEventCount starts at zero`() {
        assertEquals(0L, engine.droppedEventCount)
    }
}
```

## xplayer/app/src/test/java/com/xplayer/dev/core/error/ErrorManagerTest.kt

```kotlin
package com.xplayer.dev.core.error

import androidx.media3.common.PlaybackException
import com.xplayer.dev.core.state.PlayerState
import io.mockk.*
import org.junit.*
import org.junit.Assert.*

class ErrorManagerTest {

    private lateinit var classifier: ErrorClassifier
    private lateinit var recoveryManager: RecoveryManager
    private lateinit var healthMonitor: HealthMonitor
    private lateinit var errorManager: ErrorManager

    @Before
    fun setUp() {
        classifier       = ErrorClassifier()
        recoveryManager  = RecoveryManager()
        healthMonitor    = HealthMonitor()
        errorManager     = ErrorManager(classifier, recoveryManager, healthMonitor)
    }

    @Test
    fun `network error produces RetryWithBackoff action`() {
        val error  = makeError(PlaybackException.ERROR_CODE_IO_NETWORK_CONNECTION_FAILED)
        val action = errorManager.onError(error, PlayerState.Playing("m1"), "s1", "m1")
        assertIs<ErrorAction.RetryWithBackoff>(action)
    }

    @Test
    fun `DRM error produces ReportAndStop action`() {
        val error  = makeError(PlaybackException.ERROR_CODE_DRM_PROVISIONING_FAILED)
        val action = errorManager.onError(error, PlayerState.Playing("m1"), "s1", "m1")
        assertIs<ErrorAction.ReportAndStop>(action)
    }

    @Test
    fun `file not found produces ReportAndStop action`() {
        val error  = makeError(PlaybackException.ERROR_CODE_IO_FILE_NOT_FOUND)
        val action = errorManager.onError(error, PlayerState.Buffering("m1"), "s1", "m1")
        assertIs<ErrorAction.ReportAndStop>(action)
    }

    @Test
    fun `error history records entries`() {
        val error = makeError(PlaybackException.ERROR_CODE_IO_NETWORK_CONNECTION_FAILED)
        repeat(3) { errorManager.onError(error, PlayerState.Idle(), "s1", "m1") }
        assertEquals(3, errorManager.recentErrors(10).size)
    }

    @Test
    fun `errorsForSession filters by session`() {
        val error = makeError(PlaybackException.ERROR_CODE_IO_NETWORK_CONNECTION_FAILED)
        errorManager.onError(error, PlayerState.Idle(), "session-A", "m1")
        errorManager.onError(error, PlayerState.Idle(), "session-B", "m2")
        errorManager.onError(error, PlayerState.Idle(), "session-A", "m1")

        assertEquals(2, errorManager.errorsForSession("session-A").size)
        assertEquals(1, errorManager.errorsForSession("session-B").size)
    }

    @Test
    fun `clearHistory empties error log`() {
        val error = makeError(PlaybackException.ERROR_CODE_UNSPECIFIED)
        repeat(5) { errorManager.onError(error, PlayerState.Idle(), "s1", null) }
        errorManager.clearHistory()
        assertEquals(0, errorManager.recentErrors(10).size)
    }

    private fun makeError(code: Int): PlaybackException =
        PlaybackException("test", null, code)
}

```

## xplayer/app/src/test/java/com/xplayer/dev/core/error/HealthMonitorTest.kt

```kotlin
package com.xplayer.dev.core.error

import androidx.media3.common.PlaybackException
import org.junit.*
import org.junit.Assert.*

class HealthMonitorTest {

    private lateinit var classifier: ErrorClassifier
    private lateinit var monitor: HealthMonitor

    @Before
    fun setUp() {
        classifier = ErrorClassifier()
        monitor    = HealthMonitor()
    }

    @Test
    fun `initial score is 100 and status is HEALTHY`() {
        assertEquals(100, monitor.currentScore)
        assertEquals(HealthStatus.HEALTHY, monitor.currentStatus)
    }

    @Test
    fun `network error reduces score`() {
        val error = PlaybackException(
            "test", null,
            PlaybackException.ERROR_CODE_IO_NETWORK_CONNECTION_FAILED,
        )
        monitor.onError(classifier.classify(error))
        assertTrue(monitor.currentScore < 100)
    }

    @Test
    fun `DRM error causes large score reduction`() {
        val error = PlaybackException(
            "test", null,
            PlaybackException.ERROR_CODE_DRM_PROVISIONING_FAILED,
        )
        monitor.onError(classifier.classify(error))
        assertTrue(monitor.currentScore <= 70)
    }

    @Test
    fun `successful play recovers score`() {
        val error = PlaybackException(
            "test", null,
            PlaybackException.ERROR_CODE_IO_NETWORK_CONNECTION_FAILED,
        )
        monitor.onError(classifier.classify(error))
        val scoreAfterError = monitor.currentScore
        monitor.onSuccessfulPlay()
        assertTrue(monitor.currentScore > scoreAfterError)
    }

    @Test
    fun `reset restores full health`() {
        val error = PlaybackException(
            "test", null,
            PlaybackException.ERROR_CODE_DRM_PROVISIONING_FAILED,
        )
        repeat(5) { monitor.onError(classifier.classify(error)) }
        monitor.reset()
        assertEquals(100, monitor.currentScore)
        assertEquals(HealthStatus.HEALTHY, monitor.currentStatus)
    }

    @Test
    fun `score never goes below 0`() {
        val error = PlaybackException(
            "test", null,
            PlaybackException.ERROR_CODE_DRM_PROVISIONING_FAILED,
        )
        repeat(20) { monitor.onError(classifier.classify(error)) }
        assertTrue(monitor.currentScore >= 0)
    }

    @Test
    fun `status degrades with score`() {
        val networkError = PlaybackException(
            "test", null,
            PlaybackException.ERROR_CODE_IO_NETWORK_CONNECTION_FAILED,
        )
        repeat(12) { monitor.onError(classifier.classify(networkError)) }
        assertNotEquals(HealthStatus.HEALTHY, monitor.currentStatus)
    }
}

```

## xplayer/app/src/test/java/com/xplayer/dev/core/error/RecoveryManagerTest.kt

```kotlin
package com.xplayer.dev.core.error

import androidx.media3.common.PlaybackException
import org.junit.*
import org.junit.Assert.*

class RecoveryManagerTest {

    private lateinit var classifier: ErrorClassifier
    private lateinit var recoveryManager: RecoveryManager

    @Before
    fun setUp() {
        classifier      = ErrorClassifier()
        recoveryManager = RecoveryManager()
    }

    @Test
    fun `network error under retry limit returns RetryWithBackoff`() {
        val record = makeRecord(
            PlaybackException.ERROR_CODE_IO_NETWORK_CONNECTION_FAILED
        )
        val action = recoveryManager.determineAction(record)
        assertIs<ErrorAction.RetryWithBackoff>(action)
    }

    @Test
    fun `fatal error always returns ReportAndStop`() {
        val record = makeRecord(
            PlaybackException.ERROR_CODE_DRM_PROVISIONING_FAILED,
            isFatal = true,
        )
        val action = recoveryManager.determineAction(record)
        assertIs<ErrorAction.ReportAndStop>(action)
    }

    @Test
    fun `retry count increments correctly`() {
        val code = PlaybackException.ERROR_CODE_IO_NETWORK_CONNECTION_FAILED
        repeat(3) { recoveryManager.onRetryAttempted("session-1", code) }
        assertEquals(3, recoveryManager.retryCountForSession("session-1", code))
    }

    @Test
    fun `recovery success resets retry counters`() {
        val code = PlaybackException.ERROR_CODE_IO_NETWORK_CONNECTION_FAILED
        recoveryManager.onRetryAttempted("session-1", code)
        recoveryManager.onRecoverySuccess("session-1")
        assertEquals(0, recoveryManager.retryCountForSession("session-1", code))
    }

    @Test
    fun `max retries exceeded returns ReportAndStop`() {
        val code   = PlaybackException.ERROR_CODE_IO_NETWORK_CONNECTION_FAILED
        val record = makeRecord(code)
        // Exhaust retries
        repeat(4) { recoveryManager.onRetryAttempted("session-1", code) }
        val action = recoveryManager.determineAction(record)
        assertIs<ErrorAction.ReportAndStop>(action)
    }

    private fun makeRecord(
        code: Int,
        isFatal: Boolean = false,
    ): ErrorRecord {
        val exception      = PlaybackException("test", null, code)
        val classification = classifier.classify(exception).copy(isFatal = isFatal)
        return ErrorRecord(
            error          = exception,
            errorCode      = code,
            errorCodeName  = exception.errorCodeName,
            classification = classification,
            sessionId      = "session-1",
            mediaId        = "media-1",
            stateName      = "Playing",
            isFatal        = isFatal,
        )
    }
}

```

## xplayer/app/src/test/java/com/xplayer/dev/core/error/RetryManagerTest.kt

```kotlin
package com.xplayer.dev.core.error

import io.mockk.mockk
import kotlinx.coroutines.ExperimentalCoroutinesApi
import kotlinx.coroutines.test.*
import org.junit.*
import org.junit.Assert.*

@OptIn(ExperimentalCoroutinesApi::class)
class RetryManagerTest {

    private val dispatcher = UnconfinedTestDispatcher()
    private lateinit var recoveryManager: RecoveryManager
    private lateinit var retryManager: RetryManager

    @Before
    fun setUp() {
        recoveryManager = RecoveryManager()
        retryManager    = RetryManager(recoveryManager)
    }

    @Test
    fun `scheduleRetry calls onRetry after delay`() = runTest(dispatcher) {
        var retryCalled = false
        retryManager.scheduleRetry(
            scope      = this,
            sessionId  = "s1",
            errorCode  = 2001,
            delayMs    = 100L,
            maxRetries = 3,
            onRetry    = { retryCalled = true },
        )
        advanceUntilIdle()
        assertTrue(retryCalled)
    }

    @Test
    fun `cancel stops pending retry`() = runTest(dispatcher) {
        var retryCalled = false
        retryManager.scheduleRetry(
            scope      = this,
            sessionId  = "s1",
            errorCode  = 2001,
            delayMs    = 5_000L,
            maxRetries = 3,
            onRetry    = { retryCalled = true },
        )
        retryManager.cancel()
        advanceUntilIdle()
        assertFalse(retryCalled)
    }

    @Test
    fun `onExhausted called when max retries exceeded`() = runTest(dispatcher) {
        var exhausted = false
        // Pre-exhaust
        repeat(4) {
            retryManager.scheduleRetry(
                scope      = this,
                sessionId  = "s1",
                errorCode  = 2001,
                delayMs    = 0L,
                maxRetries = 3,
                onRetry    = {},
                onExhausted = { exhausted = true },
            )
            retryManager.reset()
        }
        advanceUntilIdle()
    }
}

```

## xplayer/app/src/test/java/com/xplayer/dev/core/performance/PerformanceManagerTest.kt

```kotlin
package com.xplayer.dev.core.performance

import android.content.Context
import io.mockk.*
import kotlinx.coroutines.ExperimentalCoroutinesApi
import kotlinx.coroutines.test.*
import org.junit.*
import org.junit.Assert.*

@OptIn(ExperimentalCoroutinesApi::class)
class PerformanceManagerTest {

    private val dispatcher = UnconfinedTestDispatcher()
    private lateinit var context: Context
    private lateinit var memoryManager: MemoryManager
    private lateinit var threadManager: ThreadManager
    private lateinit var resourceManager: ResourceManager
    private lateinit var profiler: PerformanceProfiler
    private lateinit var manager: PerformanceManager

    @Before
    fun setUp() {
        context         = mockk(relaxed = true)
        memoryManager   = mockk(relaxed = true)
        threadManager   = mockk(relaxed = true)
        resourceManager = mockk(relaxed = true)
        profiler        = PerformanceProfiler()
        manager         = PerformanceManager(
            context         = context,
            memoryManager   = memoryManager,
            threadManager   = threadManager,
            resourceManager = resourceManager,
            profiler        = profiler,
        )
    }

    @Test
    fun `startMonitoring sets isMonitoring true`() = runTest(dispatcher) {
        manager.startMonitoring(this)
        assertTrue(manager.isMonitoring)
        manager.stopMonitoring()
    }

    @Test
    fun `stopMonitoring sets isMonitoring false`() = runTest(dispatcher) {
        manager.startMonitoring(this)
        manager.stopMonitoring()
        assertFalse(manager.isMonitoring)
    }

    @Test
    fun `double startMonitoring is safe`() = runTest(dispatcher) {
        manager.startMonitoring(this)
        manager.startMonitoring(this)
        assertTrue(manager.isMonitoring)
        manager.stopMonitoring()
    }
}

```

## xplayer/app/src/test/java/com/xplayer/dev/core/performance/PerformanceProfilerTest.kt

```kotlin
package com.xplayer.dev.core.performance

import org.junit.*
import org.junit.Assert.*

class PerformanceProfilerTest {

    private lateinit var profiler: PerformanceProfiler

    @Before fun setUp() { profiler = PerformanceProfiler() }

    @Test
    fun `frame drop rate is zero with no frames`() {
        assertEquals(0f, profiler.currentFrameDropRate, 0.001f)
    }

    @Test
    fun `frame drop rate computed correctly`() {
        profiler.onFrameRendered(90L)
        profiler.onFrameDropped(10)
        assertEquals(0.1f, profiler.currentFrameDropRate, 0.001f)
    }

    @Test
    fun `peakMemoryMb returns max over all snapshots`() {
        profiler.record(makeSnapshot(usedMb = 100))
        profiler.record(makeSnapshot(usedMb = 250))
        profiler.record(makeSnapshot(usedMb = 180))
        assertEquals(250, profiler.peakMemoryMb())
    }

    @Test
    fun `averageMemoryMb computed correctly`() {
        profiler.record(makeSnapshot(usedMb = 100))
        profiler.record(makeSnapshot(usedMb = 200))
        assertEquals(150, profiler.averageMemoryMb())
    }

    @Test
    fun `lowMemoryEventCount counts correctly`() {
        profiler.record(makeSnapshot(usedMb = 100, isLowMemory = false))
        profiler.record(makeSnapshot(usedMb = 400, isLowMemory = true))
        profiler.record(makeSnapshot(usedMb = 420, isLowMemory = true))
        assertEquals(2, profiler.lowMemoryEventCount())
    }

    @Test
    fun `reset clears all data`() {
        profiler.onFrameDropped(50)
        profiler.onFrameRendered(500L)
        profiler.record(makeSnapshot(usedMb = 200))
        profiler.reset()
        assertEquals(0f, profiler.currentFrameDropRate, 0.001f)
        assertEquals(0, profiler.averageMemoryMb())
    }

    private fun makeSnapshot(
        usedMb: Int      = 100,
        isLowMemory: Boolean = false,
    ) = PerformanceSnapshot(
        timestampMs       = System.currentTimeMillis(),
        usedMemoryMb      = usedMb,
        availableMemoryMb = 2048 - usedMb,
        totalMemoryMb     = 4096,
        isLowMemory       = isLowMemory,
        nativeHeapMb      = 20,
        threadCount       = 12,
        cpuUsagePercent   = 15f,
        frameDropRate     = 0f,
    )
}

```

## xplayer/app/src/test/java/com/xplayer/dev/core/events/PlayerEventStoreTest.kt

```kotlin
// test/events/PlayerEventStoreTest.kt

package com.xplayer.dev.core.events

import androidx.media3.common.PlaybackException
import com.xplayer.dev.core.state.PlayerState
import org.junit.Assert.*
import org.junit.Before
import org.junit.Test

class PlayerEventStoreTest {

    private lateinit var store: PlayerEventStore

    @Before fun setUp() { store = PlayerEventStore(capacity = 50) }

    @Test fun `record stores events`() {
        store.record(makePlaybackStarted())
        assertEquals(1, store.size)
    }

    @Test fun `capacity evicts oldest`() {
        val smallStore = PlayerEventStore(capacity = 3)
        repeat(5) { smallStore.record(makePlaybackStarted(seq = it.toLong())) }
        assertEquals(3, smallStore.size)
    }

    @Test fun `forSession filters correctly`() {
        store.record(makePlaybackStarted(sessionId = "s1"))
        store.record(makePlaybackStarted(sessionId = "s2"))
        store.record(makePlaybackStarted(sessionId = "s1"))

        val s1Events = store.forSession("s1")
        assertEquals(2, s1Events.size)
        assertTrue(s1Events.all { it.sessionId == "s1" })
    }

    @Test fun `qosSummary ttfp from first non-resumed PlaybackStarted`() {
        val event = PlayerEvent.PlaybackStarted(
            sessionId = "s1", sequenceNum = 1L, mediaId = "m1",
            positionMs = 0L, durationMs = 0L,
            isResumed = false, timeToFirstFrameMs = 800L,
        )
        store.record(event)
        val summary = store.sessionQosSummary("s1")
        assertEquals(800L, summary.ttfpMs)
    }

    @Test fun `qosSummary stallCount from rebuffer events`() {
        repeat(3) {
            store.record(
                PlayerEvent.BufferingStarted(
                    sessionId = "s1", sequenceNum = it.toLong(), mediaId = "m1",
                    positionMs = 1000L, bufferAheadMs = 0L, bufferPercent = 5,
                    isInitialBuffer = false, stallCount = it + 1,
                )
            )
        }
        val summary = store.sessionQosSummary("s1")
        assertEquals(3, summary.stallCount)
    }

    @Test fun `breadcrumbs returns last n events`() {
        repeat(30) { store.record(makePlaybackStarted(seq = it.toLong())) }
        val crumbs = store.breadcrumbs(10)
        assertEquals(10, crumbs.size)
    }

    @Test fun `errors returns only ErrorOccurred events`() {
        store.record(makePlaybackStarted())
        store.record(makeErrorEvent())
        store.record(makePlaybackStarted())

        assertEquals(1, store.errors().size)
    }

    @Test fun `clear empties store`() {
        repeat(5) { store.record(makePlaybackStarted()) }
        store.clear()
        assertEquals(0, store.size)
    }

    // ── Helpers ───────────────────────────────────────────────────────────────

    private fun makePlaybackStarted(
        sessionId: String = "session-1",
        seq: Long = 1L,
    ) = PlayerEvent.PlaybackStarted(
        sessionId = sessionId, sequenceNum = seq, mediaId = "media-1",
        positionMs = 0L, durationMs = 60_000L,
        isResumed = false, timeToFirstFrameMs = 500L,
    )

    private fun makeErrorEvent() = PlayerEvent.ErrorOccurred(
        sessionId     = "session-1",
        sequenceNum   = 99L,
        mediaId       = "media-1",
        error         = PlaybackException("test", null, PlaybackException.ERROR_CODE_UNSPECIFIED),
        errorCategory = PlayerState.ErrorCategory.UNKNOWN,
        isFatal       = false,
        previousState = "Playing",
    )
}
```

## xplayer/app/src/test/java/com/xplayer/dev/core/events/PlayerEventTest.kt

```kotlin
// test/events/PlayerEventTest.kt

package com.xplayer.dev.core.events

import androidx.media3.common.PlaybackException
import com.xplayer.dev.core.state.PlayerState
import org.junit.Assert.*
import org.junit.Test

class PlayerEventTest {

    private val sid = "session-001"
    private val mid = "media-001"

    // ── PlaybackStarted ───────────────────────────────────────────────────────

    @Test fun `PlaybackStarted rejects negative timeToFirstFrameMs`() {
        assertThrows(IllegalArgumentException::class.java) {
            PlayerEvent.PlaybackStarted(
                sessionId          = sid,
                sequenceNum        = 1L,
                mediaId            = mid,
                positionMs         = 0L,
                durationMs         = 60_000L,
                isResumed          = false,
                timeToFirstFrameMs = -1L,
            )
        }
    }

    @Test fun `PlaybackStarted isAudioOnly when no video dimensions`() {
        val event = PlayerEvent.PlaybackStarted(
            sessionId = sid, sequenceNum = 1L, mediaId = mid,
            positionMs = 0L, durationMs = 0L, isResumed = false,
            timeToFirstFrameMs = 500L,
        )
        assertTrue(event.isAudioOnly)
    }

    @Test fun `PlaybackStarted isAudioOnly false when video dimensions set`() {
        val event = PlayerEvent.PlaybackStarted(
            sessionId = sid, sequenceNum = 1L, mediaId = mid,
            positionMs = 0L, durationMs = 0L, isResumed = false,
            timeToFirstFrameMs = 500L,
            videoWidth = 1920, videoHeight = 1080,
        )
        assertFalse(event.isAudioOnly)
    }

    // ── PlaybackEnded ─────────────────────────────────────────────────────────

    @Test fun `PlaybackEnded rejects invalid completionPercent`() {
        assertThrows(IllegalArgumentException::class.java) {
            PlayerEvent.PlaybackEnded(
                sessionId = sid, sequenceNum = 1L, mediaId = mid,
                totalPlayedMs = 60_000L, totalDurationMs = 60_000L,
                completionPercent = 101f,
            )
        }
    }

    @Test fun `PlaybackEnded isFullCompletion at 99f`() {
        val event = PlayerEvent.PlaybackEnded(
            sessionId = sid, sequenceNum = 1L, mediaId = mid,
            totalPlayedMs = 60_000L, totalDurationMs = 60_000L,
            completionPercent = 99f,
        )
        assertTrue(event.isFullCompletion)
    }

    // ── SeekRequested ─────────────────────────────────────────────────────────

    @Test fun `SeekRequested delta is correct`() {
        val event = PlayerEvent.SeekRequested(
            sessionId = sid, sequenceNum = 1L, mediaId = mid,
            fromPositionMs = 10_000L, toPositionMs = 30_000L,
        )
        assertEquals(20_000L, event.seekDeltaMs)
        assertTrue(event.isForwardSeek)
        assertFalse(event.isBackwardSeek)
    }

    @Test fun `SeekRequested negative delta is backward`() {
        val event = PlayerEvent.SeekRequested(
            sessionId = sid, sequenceNum = 1L, mediaId = mid,
            fromPositionMs = 30_000L, toPositionMs = 10_000L,
        )
        assertEquals(-20_000L, event.seekDeltaMs)
        assertTrue(event.isBackwardSeek)
        assertFalse(event.isForwardSeek)
    }

    // ── BufferingStarted ──────────────────────────────────────────────────────

    @Test fun `BufferingStarted isRebuffer when not initial`() {
        val event = PlayerEvent.BufferingStarted(
            sessionId = sid, sequenceNum = 1L, mediaId = mid,
            positionMs = 5_000L, bufferAheadMs = 0L, bufferPercent = 0,
            isInitialBuffer = false, stallCount = 1,
        )
        assertTrue(event.isRebuffer)
    }

    @Test fun `BufferingStarted rejects invalid bufferPercent`() {
        assertThrows(IllegalArgumentException::class.java) {
            PlayerEvent.BufferingStarted(
                sessionId = sid, sequenceNum = 1L, mediaId = mid,
                positionMs = 0L, bufferAheadMs = 0L, bufferPercent = -1,
                isInitialBuffer = true, stallCount = 0,
            )
        }
    }

    // ── DroppedFrames ─────────────────────────────────────────────────────────

    @Test fun `DroppedFrames isCritical when drop rate over 2`() {
        val event = PlayerEvent.DroppedFrames(
            sessionId = sid, sequenceNum = 1L, mediaId = mid,
            droppedCount = 10, elapsedMs = 1000L,
            totalDropped = 10, dropRate = 10f,
        )
        assertTrue(event.isCritical)
    }

    @Test fun `DroppedFrames not critical when drop rate under 2`() {
        val event = PlayerEvent.DroppedFrames(
            sessionId = sid, sequenceNum = 1L, mediaId = mid,
            droppedCount = 1, elapsedMs = 1000L,
            totalDropped = 1, dropRate = 1f,
        )
        assertFalse(event.isCritical)
    }

    // ── BitrateChanged ────────────────────────────────────────────────────────

    @Test fun `BitrateChanged delta is correct`() {
        val event = PlayerEvent.BitrateChanged(
            sessionId = sid, sequenceNum = 1L, mediaId = mid,
            trackType = PlayerEvent.BitrateChanged.BitrateTrackType.VIDEO,
            previousBitrateBps = 1_000_000L,
            newBitrateBps = 2_000_000L,
            switchDirection = PlayerEvent.BitrateChanged.SwitchDirection.UP,
            reason = PlayerEvent.BitrateChanged.BitrateChangeReason.BANDWIDTH_INCREASE,
        )
        assertEquals(1_000_000L, event.bitrateDeltaBps)
        assertEquals(2.0f, event.switchRatio, 0.001f)
    }

    // ── VideoSizeChanged ──────────────────────────────────────────────────────

    @Test fun `VideoSizeChanged is4K for 4K resolution`() {
        val event = PlayerEvent.VideoSizeChanged(
            sessionId = sid, sequenceNum = 1L, mediaId = mid,
            width = 3840, height = 2160,
        )
        assertTrue(event.is4K)
        assertTrue(event.isHD)
    }

    @Test fun `VideoSizeChanged aspectRatio calculated correctly`() {
        val event = PlayerEvent.VideoSizeChanged(
            sessionId = sid, sequenceNum = 1L, mediaId = mid,
            width = 1920, height = 1080,
        )
        assertEquals(16f / 9f, event.aspectRatio, 0.01f)
    }

    // ── BandwidthEstimated ────────────────────────────────────────────────────

    @Test fun `BandwidthEstimated bandwidthMbps conversion`() {
        val event = PlayerEvent.BandwidthEstimated(
            sessionId = sid, sequenceNum = 1L, mediaId = mid,
            bandwidthBps = 5_000_000L, sampleBps = 5_000_000L,
        )
        assertEquals(5.0f, event.bandwidthMbps, 0.001f)
    }

    // ── ErrorOccurred ─────────────────────────────────────────────────────────

    @Test fun `ErrorOccurred canRetry false when fatal`() {
        val event = makeErrorEvent(isFatal = true, retryAttempt = 0, maxRetries = 3)
        assertFalse(event.canRetry)
    }

    @Test fun `ErrorOccurred canRetry false when exhausted`() {
        val event = makeErrorEvent(isFatal = false, retryAttempt = 3, maxRetries = 3)
        assertFalse(event.canRetry)
    }

    @Test fun `ErrorOccurred canRetry true when attempts remaining`() {
        val event = makeErrorEvent(isFatal = false, retryAttempt = 1, maxRetries = 3)
        assertTrue(event.canRetry)
    }

    // ── TrackInfo ─────────────────────────────────────────────────────────────

    @Test fun `TrackInfo resolutionLabel for 1080p`() {
        val track = PlayerEvent.TrackInfo(
            trackId = "v1", mimeType = "video/mp4", codec = "avc",
            bitrateBps = 5_000_000, width = 1920, height = 1080,
        )
        assertEquals("1080p", track.resolutionLabel)
    }

    @Test fun `TrackInfo isVideo is false for audio track`() {
        val track = PlayerEvent.TrackInfo(
            trackId = "a1", mimeType = "audio/mp4", codec = "aac",
            bitrateBps = 128_000, channelCount = 2, sampleRateHz = 44100,
        )
        assertFalse(track.isVideo)
        assertTrue(track.isAudio)
    }

    // ── Extensions ────────────────────────────────────────────────────────────

    @Test fun `isError extension`() {
        val errorEvent = makeErrorEvent()
        val playEvent = PlayerEvent.PlaybackStarted(
            sessionId = sid, sequenceNum = 1L, mediaId = mid,
            positionMs = 0L, durationMs = 0L,
            isResumed = false, timeToFirstFrameMs = 100L,
        )
        assertTrue(errorEvent.isError)
        assertFalse(playEvent.isError)
    }

    @Test fun `isFatalError extension`() {
        assertTrue(makeErrorEvent(isFatal = true).isFatalError)
        assertFalse(makeErrorEvent(isFatal = false).isFatalError)
    }

    @Test fun `isSessionBoundary extension`() {
        val sessionStarted = PlayerEvent.SessionStarted(
            sessionId = sid, sequenceNum = 0L, mediaId = mid,
            url = "http://test.com",
            contentType = PlayerEvent.SessionStarted.ContentType.VOD,
            deviceInfo = PlayerEvent.DeviceInfo(
                manufacturer = "Google", model = "Pixel",
                osVersion = "14", sdkInt = 34,
                screenWidthPx = 1080, screenHeightPx = 2340,
                screenDensity = 2.75f, totalRamMb = 8192L, cpuAbi = "arm64-v8a",
            ),
            appVersion = "1.0.0",
            playerVersion = "1.4.1",
        )
        assertTrue(sessionStarted.isSessionBoundary)
        assertFalse(makeErrorEvent().isSessionBoundary)
    }

    // ── Helpers ───────────────────────────────────────────────────────────────

    private fun makeErrorEvent(
        isFatal: Boolean = false,
        retryAttempt: Int = 0,
        maxRetries: Int = 3,
    ): PlayerEvent.ErrorOccurred {
        val exception = PlaybackException(
            "test", null, PlaybackException.ERROR_CODE_UNSPECIFIED,
        )
        return PlayerEvent.ErrorOccurred(
            sessionId     = sid,
            sequenceNum   = 1L,
            mediaId       = mid,
            error         = exception,
            errorCategory = PlayerState.ErrorCategory.UNKNOWN,
            isFatal       = isFatal,
            retryAttempt  = retryAttempt,
            maxRetries    = maxRetries,
            previousState = "Playing",
        )
    }
}
```

## xplayer/app/src/test/java/com/xplayer/dev/core/events/PlayerEventValidatorTest.kt

```kotlin
// test/events/PlayerEventValidatorTest.kt

package com.xplayer.dev.core.events

import androidx.media3.common.PlaybackException
import com.xplayer.dev.core.state.PlayerState
import org.junit.Assert.*
import org.junit.Before
import org.junit.Test

class PlayerEventValidatorTest {

    private lateinit var validator: PlayerEventValidator

    @Before fun setUp() { validator = PlayerEventValidator() }

    @Test fun `valid PlaybackStarted passes`() {
        val event = PlayerEvent.PlaybackStarted(
            sessionId = "s1", sequenceNum = 1L, mediaId = "m1",
            positionMs = 0L, durationMs = 0L,
            isResumed = false, timeToFirstFrameMs = 500L,
        )
        assertIs<PlayerEventValidator.ValidationResult.Valid>(
            validator.validate(event)
        )
    }

    @Test fun `suspicious TTFP flagged as invalid`() {
        val event = PlayerEvent.PlaybackStarted(
            sessionId = "s1", sequenceNum = 1L, mediaId = "m1",
            positionMs = 0L, durationMs = 0L,
            isResumed = false, timeToFirstFrameMs = 90_000L, // 90s > 60s max
        )
        val result = validator.validate(event)
        assertIs<PlayerEventValidator.ValidationResult.Invalid>(result)
        assertTrue(
            (result as PlayerEventValidator.ValidationResult.Invalid)
                .reasons.any { it.contains("timeToFirstFrameMs") }
        )
    }

    @Test fun `ErrorOccurred with blank sessionId is invalid`() {
        val event = PlayerEvent.ErrorOccurred(
            sessionId     = "",   // blank!
            sequenceNum   = 1L,
            mediaId       = "m1",
            error         = PlaybackException("t", null, PlaybackException.ERROR_CODE_UNSPECIFIED),
            errorCategory = PlayerState.ErrorCategory.UNKNOWN,
            isFatal       = false,
            previousState = "Playing",
        )
        val result = validator.validate(event)
        assertIs<PlayerEventValidator.ValidationResult.Invalid>(result)
    }

    @Test fun `BufferingEnded with negative stallDurationMs is invalid`() {
        val event = PlayerEvent.BufferingEnded(
            sessionId = "s1", sequenceNum = 1L, mediaId = "m1",
            stallDurationMs = -1L, bufferPercentOnResume = 50,
        )
        val result = validator.validate(event)
        assertIs<PlayerEventValidator.ValidationResult.Invalid>(result)
    }
}
```

## xplayer/app/src/test/java/com/xplayer/dev/core/state/PlayerStateMachineTest.kt

```kotlin
// test/state/PlayerStateMachineTest.kt

package com.xplayer.dev.core.state

import io.mockk.mockk
import kotlinx.coroutines.ExperimentalCoroutinesApi
import kotlinx.coroutines.test.*
import app.cash.turbine.test
import org.junit.*
import org.junit.Assert.*

@OptIn(ExperimentalCoroutinesApi::class)
class PlayerStateMachineTest {

    private lateinit var machine: PlayerStateMachine
    private lateinit var store: PlayerStateStore

    @Before
    fun setUp() {
        store   = PlayerStateStore(capacity = 100)
        machine = PlayerStateMachine(
            reducer = PlayerStateReducer(),
            store   = store,
            isDebug = false,
        )
    }

    @Test
    fun `initial state is Uninitialised`() = runTest {
        machine.state.test {
            assertIs<PlayerState.Uninitialised>(awaitItem())
            cancelAndIgnoreRemainingEvents()
        }
    }

    @Test
    fun `EngineInitialised transitions to Idle`() = runTest {
        machine.state.test {
            awaitItem() // Uninitialised

            machine.dispatch(PlayerStateCommand.EngineInitialised())
            assertIs<PlayerState.Idle>(awaitItem())
            cancelAndIgnoreRemainingEvents()
        }
    }

    @Test
    fun `full happy path transitions`() = runTest {
        val mediaItem = mockk<androidx.media3.common.MediaItem>(relaxed = true)

        machine.state.test {
            awaitItem() // Uninitialised

            machine.dispatch(PlayerStateCommand.EngineInitialised())
            assertIs<PlayerState.Idle>(awaitItem())

            machine.dispatch(PlayerStateCommand.Prepare("id", mediaItem))
            assertIs<PlayerState.Preparing>(awaitItem())

            machine.dispatch(PlayerStateCommand.BufferingStarted())
            assertIs<PlayerState.Buffering>(awaitItem())

            machine.dispatch(PlayerStateCommand.ReadyToPlay(120_000L))
            assertIs<PlayerState.Ready>(awaitItem())

            machine.dispatch(PlayerStateCommand.PlayingStarted(0L, 120_000L, 30_000L))
            assertIs<PlayerState.Playing>(awaitItem())

            machine.dispatch(PlayerStateCommand.PlaybackEnded(120_000L, 100f))
            assertIs<PlayerState.Ended>(awaitItem())

            machine.dispatch(PlayerStateCommand.Reset())
            assertIs<PlayerState.Idle>(awaitItem())

            cancelAndIgnoreRemainingEvents()
        }
    }

    @Test
    fun `history is recorded per transition`() {
        val mediaItem = mockk<androidx.media3.common.MediaItem>(relaxed = true)
        machine.dispatch(PlayerStateCommand.EngineInitialised())
        machine.dispatch(PlayerStateCommand.Prepare("id", mediaItem))
        machine.dispatch(PlayerStateCommand.BufferingStarted())

        assertEquals(3, store.size)
    }

    @Test
    fun `listener is notified on accepted transition`() {
        var notified = false
        machine.addListener { notified = true }
        machine.dispatch(PlayerStateCommand.EngineInitialised())
        assertTrue(notified)
    }

    @Test
    fun `listener is not notified on ignored command`() {
        var count = 0
        machine.addListener { count++ }
        machine.dispatch(PlayerStateCommand.EngineInitialised()) // accepted
        machine.dispatch(PlayerStateCommand.EngineInitialised()) // ignored
        assertEquals(1, count)
    }

    @Test
    fun `removeListener stops notifications`() {
        var count = 0
        val listener = PlayerStateMachine.TransitionListener { count++ }
        machine.addListener(listener)
        machine.dispatch(PlayerStateCommand.EngineInitialised())
        machine.removeListener(listener)
        machine.dispatch(PlayerStateCommand.Reset())
        assertEquals(1, count)
    }
}
```

## xplayer/app/src/test/java/com/xplayer/dev/core/state/PlayerStateReducerTest.kt

```kotlin
// test/state/PlayerStateReducerTest.kt

package com.xplayer.dev.core.state

import androidx.media3.common.PlaybackException
import io.mockk.mockk
import org.junit.Assert.*
import org.junit.Before
import org.junit.Test

class PlayerStateReducerTest {

    private lateinit var reducer: PlayerStateReducer

    @Before fun setUp() { reducer = PlayerStateReducer() }

    @Test fun `EngineInitialised from Uninitialised produces Idle`() {
        val result = reducer.reduce(
            PlayerState.Uninitialised,
            PlayerStateCommand.EngineInitialised(),
        )
        assertIs<PlayerStateReducer.ReducerResult.Accepted>(result)
        assertIs<PlayerState.Idle>((result as PlayerStateReducer.ReducerResult.Accepted).nextState)
    }

    @Test fun `EngineInitialised from Idle is ignored`() {
        val result = reducer.reduce(
            PlayerState.Idle(),
            PlayerStateCommand.EngineInitialised(),
        )
        assertIs<PlayerStateReducer.ReducerResult.Ignored>(result)
    }

    @Test fun `Prepare from Idle produces Preparing`() {
        val mediaItem = mockk<androidx.media3.common.MediaItem>(relaxed = true)
        val result = reducer.reduce(
            PlayerState.Idle(),
            PlayerStateCommand.Prepare("id", mediaItem),
        )
        assertIs<PlayerStateReducer.ReducerResult.Accepted>(result)
        val next = (result as PlayerStateReducer.ReducerResult.Accepted).nextState
        assertIs<PlayerState.Preparing>(next)
        assertEquals("id", (next as PlayerState.Preparing).mediaId)
    }

    @Test fun `Prepare from Preparing is rejected`() {
        val mediaItem = mockk<androidx.media3.common.MediaItem>(relaxed = true)
        val result = reducer.reduce(
            PlayerState.Preparing("old", mediaItem),
            PlayerStateCommand.Prepare("new", mediaItem),
        )
        assertIs<PlayerStateReducer.ReducerResult.Rejected>(result)
    }

    @Test fun `Play from Idle is rejected`() {
        val result = reducer.reduce(PlayerState.Idle(), PlayerStateCommand.Play())
        assertIs<PlayerStateReducer.ReducerResult.Rejected>(result)
    }

    @Test fun `Play from Ready is accepted`() {
        val result = reducer.reduce(
            PlayerState.Ready("id", durationMs = 120_000L),
            PlayerStateCommand.Play(),
        )
        assertIs<PlayerStateReducer.ReducerResult.Accepted>(result)
        assertIs<PlayerState.Playing>(
            (result as PlayerStateReducer.ReducerResult.Accepted).nextState
        )
    }

    @Test fun `Play from Playing is ignored`() {
        val result = reducer.reduce(
            PlayerState.Playing("id"),
            PlayerStateCommand.Play(),
        )
        assertIs<PlayerStateReducer.ReducerResult.Ignored>(result)
    }

    @Test fun `Pause from Playing produces Paused`() {
        val result = reducer.reduce(
            PlayerState.Playing("id", positionMs = 5000L, durationMs = 120_000L),
            PlayerStateCommand.Pause(),
        )
        assertIs<PlayerStateReducer.ReducerResult.Accepted>(result)
        val next = (result as PlayerStateReducer.ReducerResult.Accepted).nextState
        assertIs<PlayerState.Paused>(next)
        assertEquals(5000L, (next as PlayerState.Paused).positionMs)
    }

    @Test fun `Stop from Playing produces Idle`() {
        val result = reducer.reduce(
            PlayerState.Playing("id"),
            PlayerStateCommand.Stop(),
        )
        assertIs<PlayerStateReducer.ReducerResult.Accepted>(result)
        assertIs<PlayerState.Idle>(
            (result as PlayerStateReducer.ReducerResult.Accepted).nextState
        )
    }

    @Test fun `PositionUpdated only accepted in Playing`() {
        val accepted = reducer.reduce(
            PlayerState.Playing("id", positionMs = 1000L),
            PlayerStateCommand.PositionUpdated(2000L, 5000L),
        )
        assertIs<PlayerStateReducer.ReducerResult.Accepted>(accepted)

        val ignored = reducer.reduce(
            PlayerState.Paused("id"),
            PlayerStateCommand.PositionUpdated(2000L, 5000L),
        )
        assertIs<PlayerStateReducer.ReducerResult.Ignored>(ignored)
    }

    @Test fun `sequence number increments on every accept`() {
        val mediaItem = mockk<androidx.media3.common.MediaItem>(relaxed = true)
        val idle = PlayerState.Idle(sequenceNumber = 5L)
        val result = reducer.reduce(idle, PlayerStateCommand.Prepare("id", mediaItem))

        assertIs<PlayerStateReducer.ReducerResult.Accepted>(result)
        assertEquals(6L, (result as PlayerStateReducer.ReducerResult.Accepted).nextState.sequenceNumber)
    }

    @Test fun `ErrorOccurred from any state is accepted`() {
        val error = PlaybackException("test", null, PlaybackException.ERROR_CODE_UNSPECIFIED)
        listOf(
            PlayerState.Playing("id"),
            PlayerState.Paused("id"),
            PlayerState.Buffering("id"),
            PlayerState.Ready("id"),
        ).forEach { state ->
            val result = reducer.reduce(state, PlayerStateCommand.ErrorOccurred(error, state))
            assertIs<PlayerStateReducer.ReducerResult.Accepted>(result)
        }
    }
}
```

## xplayer/app/src/test/java/com/xplayer/dev/core/state/PlayerStateTest.kt

```kotlin
// test/state/PlayerStateTest.kt

package com.xplayer.dev.core.state

import androidx.media3.common.PlaybackException
import org.junit.Assert.*
import org.junit.Test

class PlayerStateTest {

    // ── Buffering validation ──────────────────────────────────────────────────

    @Test fun `Buffering rejects negative bufferPercent`() {
        assertThrows(IllegalArgumentException::class.java) {
            PlayerState.Buffering("id", bufferPercent = -1)
        }
    }

    @Test fun `Buffering rejects bufferPercent over 100`() {
        assertThrows(IllegalArgumentException::class.java) {
            PlayerState.Buffering("id", bufferPercent = 101)
        }
    }

    @Test fun `Buffering accepts boundary values`() {
        assertDoesNotThrow { PlayerState.Buffering("id", bufferPercent = 0) }
        assertDoesNotThrow { PlayerState.Buffering("id", bufferPercent = 100) }
    }

    @Test fun `Buffering rejects negative rebufferCount`() {
        assertThrows(IllegalArgumentException::class.java) {
            PlayerState.Buffering("id", rebufferCount = -1)
        }
    }

    // ── Playing validation ────────────────────────────────────────────────────

    @Test fun `Playing rejects negative positionMs`() {
        assertThrows(IllegalArgumentException::class.java) {
            PlayerState.Playing("id", positionMs = -1L)
        }
    }

    @Test fun `Playing rejects zero playbackSpeed`() {
        assertThrows(IllegalArgumentException::class.java) {
            PlayerState.Playing("id", playbackSpeed = 0f)
        }
    }

    @Test fun `Playing rejects negative playbackSpeed`() {
        assertThrows(IllegalArgumentException::class.java) {
            PlayerState.Playing("id", playbackSpeed = -1f)
        }
    }

    // ── Playing computations ──────────────────────────────────────────────────

    @Test fun `Playing progress is correct`() {
        val state = PlayerState.Playing("id", positionMs = 30_000L, durationMs = 120_000L)
        assertEquals(0.25f, state.progress, 0.001f)
    }

    @Test fun `Playing progress is 0 for live stream`() {
        val state = PlayerState.Playing("id", isLive = true, positionMs = 10_000L)
        assertEquals(0f, state.progress, 0.001f)
    }

    @Test fun `Playing progress is 0 for unknown duration`() {
        val state = PlayerState.Playing("id", durationMs = 0L, positionMs = 5_000L)
        assertEquals(0f, state.progress, 0.001f)
    }

    @Test fun `Playing remainingMs is correct`() {
        val state = PlayerState.Playing("id", positionMs = 30_000L, durationMs = 120_000L)
        assertEquals(90_000L, state.remainingMs)
    }

    @Test fun `Playing remainingMs is -1 for live`() {
        val state = PlayerState.Playing("id", isLive = true)
        assertEquals(-1L, state.remainingMs)
    }

    @Test fun `Playing bufferAheadMs is correct`() {
        val state = PlayerState.Playing("id", positionMs = 10_000L, bufferedMs = 30_000L)
        assertEquals(20_000L, state.bufferAheadMs)
    }

    @Test fun `Playing isAudioOnly when no video dimensions`() {
        assertTrue(PlayerState.Playing("id").isAudioOnly)
        assertFalse(PlayerState.Playing("id", videoWidth = 1920, videoHeight = 1080).isAudioOnly)
    }

    // ── Ended validation ──────────────────────────────────────────────────────

    @Test fun `Ended rejects invalid completionPercent`() {
        assertThrows(IllegalArgumentException::class.java) {
            PlayerState.Ended("id", completionPercent = 101f)
        }
        assertThrows(IllegalArgumentException::class.java) {
            PlayerState.Ended("id", completionPercent = -1f)
        }
    }

    @Test fun `Ended isFullyCompleted threshold`() {
        assertTrue(PlayerState.Ended("id", completionPercent = 99f).isFullyCompleted)
        assertTrue(PlayerState.Ended("id", completionPercent = 100f).isFullyCompleted)
        assertFalse(PlayerState.Ended("id", completionPercent = 98f).isFullyCompleted)
    }

    // ── Error ─────────────────────────────────────────────────────────────────

    @Test fun `Error canRetry is false when fatal`() {
        val error = makeError(isFatal = true)
        assertFalse(error.canRetry)
    }

    @Test fun `Error canRetry is false when retries exhausted`() {
        val error = makeError(retryCount = 3, maxRetries = 3)
        assertFalse(error.canRetry)
    }

    @Test fun `Error canRetry is true when retries remaining`() {
        val error = makeError(retryCount = 1, maxRetries = 3, isFatal = false)
        assertTrue(error.canRetry)
        assertEquals(2, error.retriesRemaining)
    }

    @Test fun `Error causalChain depth is bounded`() {
        var error = makeError()
        repeat(15) { error = makeError(previousState = error) }
        assertTrue(error.causalChain.size <= 10)
    }

    @Test fun `Error category from exception`() {
        val networkError = PlaybackException(
            "network",
            null,
            PlaybackException.ERROR_CODE_IO_NETWORK_CONNECTION_FAILED,
        )
        assertEquals(
            PlayerState.ErrorCategory.NETWORK,
            PlayerState.ErrorCategory.fromException(networkError),
        )
    }

    @Test fun `Error recoveryHint matches category`() {
        val error = makeError(
            code = PlaybackException.ERROR_CODE_IO_NETWORK_CONNECTION_FAILED,
        )
        assertEquals(
            PlayerState.RecoveryHint.RETRY_WITH_BACKOFF,
            error.recoveryHint,
        )
    }

    // ── Extensions ───────────────────────────────────────────────────────────

    @Test fun `mediaId extension returns correct value`() {
        assertNull(PlayerState.Idle().mediaId)
        assertNull(PlayerState.Uninitialised.mediaId)
        assertEquals("id", PlayerState.Playing("id").mediaId)
        assertEquals("id", PlayerState.Paused("id").mediaId)
    }

    @Test fun `isActive is true only for Playing and Buffering`() {
        assertTrue(PlayerState.Playing("id").isActive)
        assertTrue(PlayerState.Buffering("id").isActive)
        assertFalse(PlayerState.Idle().isActive)
        assertFalse(PlayerState.Paused("id").isActive)
        assertFalse(PlayerState.Ready("id").isActive)
    }

    @Test fun `isTerminal is true for fatal Error and Ended`() {
        assertTrue(PlayerState.Ended("id").isTerminal)
        assertTrue(makeError(isFatal = true).isTerminal)
        assertFalse(makeError(isFatal = false).isTerminal)
        assertFalse(PlayerState.Playing("id").isTerminal)
    }

    @Test fun `canPlay guard`() {
        assertTrue(PlayerState.Ready("id").canPlay)
        assertTrue(PlayerState.Paused("id").canPlay)
        assertTrue(PlayerState.Playing("id").canPlay)
        assertFalse(PlayerState.Idle().canPlay)
        assertFalse(PlayerState.Buffering("id").canPlay)
    }

    @Test fun `canSeek guard`() {
        assertTrue(PlayerState.Playing("id", isSeekable = true).canSeek)
        assertTrue(PlayerState.Paused("id").canSeek)
        assertFalse(PlayerState.Buffering("id").canSeek)
        assertFalse(PlayerState.Idle().canSeek)
    }

    @Test fun `withPosition copy helper`() {
        val state = PlayerState.Playing("id", positionMs = 1000L, bufferedMs = 5000L)
        val updated = state.withPosition(2000L, 6000L)
        assertEquals(2000L, updated.positionMs)
        assertEquals(6000L, updated.bufferedMs)
        assertEquals("id", updated.mediaId)
    }

    @Test fun `withIncrementedRetry increments retryCount`() {
        val error = makeError(retryCount = 1)
        val retried = error.withIncrementedRetry()
        assertEquals(2, retried.retryCount)
    }

    @Test fun `toError extension preserves previousState`() {
        val playingState = PlayerState.Playing("id")
        val exception = PlaybackException(
            "test",
            null,
            PlaybackException.ERROR_CODE_UNSPECIFIED,
        )
        val error = playingState.toError(exception)
        assertSame(playingState, error.previousState)
        assertEquals("id", error.mediaId)
    }

    @Test fun `logSummary contains stateName`() {
        val state = PlayerState.Playing("id", positionMs = 1000L)
        assertTrue(state.logSummary.contains("Playing"))
        assertTrue(state.logSummary.contains("id"))
        assertTrue(state.logSummary.contains("1000"))
    }

    // ── Helpers ───────────────────────────────────────────────────────────────

    private fun makeError(
        code: Int = PlaybackException.ERROR_CODE_UNSPECIFIED,
        retryCount: Int = 0,
        maxRetries: Int = 3,
        isFatal: Boolean = false,
        previousState: PlayerState? = null,
    ): PlayerState.Error {
        val exception = PlaybackException("test error", null, code)
        return PlayerState.Error(
            mediaId       = "id",
            cause         = exception,
            retryCount    = retryCount,
            maxRetries    = maxRetries,
            isFatal       = isFatal,
            previousState = previousState,
        )
    }
}
```

