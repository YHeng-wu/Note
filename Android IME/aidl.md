# 1.IInputMethodManager
/frameworks/base/core/java/com/android/internal/view/IInputMethodManager.aidl
```java
/**
 * Public interface to the global input method manager, used by all client
 * applications.
 * You need to update BridgeIInputMethodManager.java as well when changing
 * this file.
 */
interface IInputMethodManager {
    // TODO: Use ParceledListSlice instead
    List<InputMethodInfo> getInputMethodList();
    List<InputMethodInfo> getVrInputMethodList();
    // TODO: Use ParceledListSlice instead
    List<InputMethodInfo> getEnabledInputMethodList();
    List<InputMethodSubtype> getEnabledInputMethodSubtypeList(in String imiId,
            boolean allowsImplicitlySelectedSubtypes);
    InputMethodSubtype getLastInputMethodSubtype();
    // TODO: We should change the return type from List to List<Parcelable>
    // Currently there is a bug that aidl doesn't accept List<Parcelable>
    List getShortcutInputMethodsAndSubtypes();
    void addClient(in IInputMethodClient client,
            in IInputContext inputContext, int uid, int pid);
    void removeClient(in IInputMethodClient client);

    void finishInput(in IInputMethodClient client);
    boolean showSoftInput(in IInputMethodClient client, int flags,
            in ResultReceiver resultReceiver);
    boolean hideSoftInput(in IInputMethodClient client, int flags,
            in ResultReceiver resultReceiver);
    // If windowToken is null, this just does startInput().  Otherwise this reports that a window
    // has gained focus, and if 'attribute' is non-null then also does startInput.
    // @NonNull
    InputBindResult startInputOrWindowGainedFocus(
            /* @InputMethodClient.StartInputReason */ int startInputReason,
            in IInputMethodClient client, in IBinder windowToken, int controlFlags,
            /* @android.view.WindowManager.LayoutParams.SoftInputModeFlags */ int softInputMode,
            int windowFlags, in EditorInfo attribute, IInputContext inputContext,
            /* @InputConnectionInspector.MissingMethodFlags */ int missingMethodFlags,
            int unverifiedTargetSdkVersion);

    void showInputMethodPickerFromClient(in IInputMethodClient client,
            int auxiliarySubtypeMode);
    void showInputMethodAndSubtypeEnablerFromClient(in IInputMethodClient client, String topId);
    boolean isInputMethodPickerShownForTest();
    void setInputMethod(in IBinder token, String id);
    void setInputMethodAndSubtype(in IBinder token, String id, in InputMethodSubtype subtype);
    void hideMySoftInput(in IBinder token, int flags);
    void showMySoftInput(in IBinder token, int flags);
    void updateStatusIcon(in IBinder token, String packageName, int iconId);
    void setImeWindowStatus(in IBinder token, in IBinder startInputToken, int vis,
            int backDisposition);
    void registerSuggestionSpansForNotification(in SuggestionSpan[] spans);
    boolean notifySuggestionPicked(in SuggestionSpan span, String originalString, int index);
    InputMethodSubtype getCurrentInputMethodSubtype();
    boolean setCurrentInputMethodSubtype(in InputMethodSubtype subtype);
    boolean switchToPreviousInputMethod(in IBinder token);
    boolean switchToNextInputMethod(in IBinder token, boolean onlyCurrentIme);
    boolean shouldOfferSwitchingToNextInputMethod(in IBinder token);
    void setAdditionalInputMethodSubtypes(String id, in InputMethodSubtype[] subtypes);
    int getInputMethodWindowVisibleHeight();
    void clearLastInputMethodWindowForTransition(in IBinder token);

    IInputContentUriToken createInputContentUriToken(in IBinder token, in Uri contentUri,
            in String packageName);

    void reportFullscreenMode(in IBinder token, boolean fullscreen);

    oneway void notifyUserAction(int sequenceNumber);
}
```
# 2.IInputMethodClient
/frameworks/base/core/java/com/android/internal/view/IInputMethodClient.aidl

```java
/**
 * Interface a client of the IInputMethodManager implements, to identify
 * itself and receive information about changes to the global manager state.
 */
oneway interface IInputMethodClient {
    void setUsingInputMethod(boolean state);
    void onBindMethod(in InputBindResult res);
    // unbindReason corresponds to InputMethodClient.UnbindReason.
    void onUnbindMethod(int sequence, int unbindReason);
    void setActive(boolean active, boolean fullscreen);
    void setUserActionNotificationSequenceNumber(int sequenceNumber);
    void reportFullscreenMode(boolean fullscreen);
}
```
# 3.IInputMethod
/frameworks/base/core/java/com/android/internal/view/IInputMethod.aidl
```java
/**
 * Top-level interface to an input method component (implemented in a
 * Service).
 * {@hide}
 */
oneway interface IInputMethod {
    void attachToken(IBinder token);

    void bindInput(in InputBinding binding);

    void unbindInput();

    void startInput(in IBinder startInputToken, in IInputContext inputContext, int missingMethods,
            in EditorInfo attribute, boolean restarting);

    void createSession(in InputChannel channel, IInputSessionCallback callback);

    void setSessionEnabled(IInputMethodSession session, boolean enabled);

    void revokeSession(IInputMethodSession session);

    void showSoftInput(int flags, in ResultReceiver resultReceiver);

    void hideSoftInput(int flags, in ResultReceiver resultReceiver);

    void changeInputMethodSubtype(in InputMethodSubtype subtype);
}
```
# 4.IInputMethodSession
/frameworks/base/core/java/com/android/internal/view/IInputMethodSession.aidl
```java
/**
 * Sub-interface of IInputMethod which is safe to give to client applications.
 * {@hide}
 */
oneway interface IInputMethodSession {
    void finishInput();

    void updateExtractedText(int token, in ExtractedText text);
    
    void updateSelection(int oldSelStart, int oldSelEnd,
            int newSelStart, int newSelEnd,
            int candidatesStart, int candidatesEnd);

    void viewClicked(boolean focusChanged);

    void updateCursor(in Rect newCursor);

    void displayCompletions(in CompletionInfo[] completions);

    void appPrivateCommand(String action, in Bundle data);

    void toggleSoftInput(int showFlags, int hideFlags);

    void finishSession();

    void updateCursorAnchorInfo(in CursorAnchorInfo cursorAnchorInfo);
}
```


# 5.IInputSessionCallback
/frameworks/base/core/java/com/android/internal/view/IInputSessionCallback.aidl
```java
oneway interface IInputSessionCallback {
    void sessionCreated(IInputMethodSession session);
}
```
# 6.IInputContext
/frameworks/base/core/java/com/android/internal/view/IInputContext.aidl
```java
/**
 * Interface from an input method to the application, allowing it to perform
 * edits on the current input field and other interactions with the application.
 * {@hide}
 */
 oneway interface IInputContext {
    void getTextBeforeCursor(int length, int flags, int seq, IInputContextCallback callback); 

    void getTextAfterCursor(int length, int flags, int seq, IInputContextCallback callback);
    
    void getCursorCapsMode(int reqModes, int seq, IInputContextCallback callback);
    
    void getExtractedText(in ExtractedTextRequest request, int flags, int seq,
            IInputContextCallback callback);

    void deleteSurroundingText(int beforeLength, int afterLength);
    void deleteSurroundingTextInCodePoints(int beforeLength, int afterLength);

    void setComposingText(CharSequence text, int newCursorPosition);

    void finishComposingText();
    
    void commitText(CharSequence text, int newCursorPosition);

    void commitCompletion(in CompletionInfo completion);

    void commitCorrection(in CorrectionInfo correction);

    void setSelection(int start, int end);
    
    void performEditorAction(int actionCode);
    
    void performContextMenuAction(int id);
    
    void beginBatchEdit();
    
    void endBatchEdit();

    void sendKeyEvent(in KeyEvent event);
    
    void clearMetaKeyStates(int states);
    
    void performPrivateCommand(String action, in Bundle data);

    void setComposingRegion(int start, int end);

    void getSelectedText(int flags, int seq, IInputContextCallback callback);

    void requestUpdateCursorAnchorInfo(int cursorUpdateMode, int seq,
            IInputContextCallback callback);

    void commitContent(in InputContentInfo inputContentInfo, int flags, in Bundle opts, int sec,
            IInputContextCallback callback);
}
```

# 8 IInputContextCallback
/frameworks/base/core/java/com/android/internal/view/IInputContextCallback.aidl
```java
/**
 * {@hide}
 */
oneway interface IInputContextCallback {
    void setTextBeforeCursor(CharSequence textBeforeCursor, int seq);
    void setTextAfterCursor(CharSequence textAfterCursor, int seq);
    void setCursorCapsMode(int capsMode, int seq);
    void setExtractedText(in ExtractedText extractedText, int seq);
    void setSelectedText(CharSequence selectedText, int seq);
    void setRequestUpdateCursorAnchorInfoResult(boolean result, int seq);
    void setCommitContentResult(boolean result, int seq);
}
```