set(GUI_SRCS
	ARCodeAddEdit.cpp
	AboutDolphin.cpp
	ConfigMain.cpp
	Cheats/CheatSearchTab.cpp
	Cheats/CheatsWindow.cpp
	Cheats/CreateCodeDialog.cpp
	Cheats/GeckoCodeDiag.cpp
	Debugger/BreakpointDlg.cpp
	Debugger/BreakpointView.cpp
	Debugger/BreakpointWindow.cpp
	Debugger/CodeView.cpp
	Debugger/CodeWindow.cpp
	Debugger/CodeWindowFunctions.cpp
	Debugger/DSPDebugWindow.cpp
	Debugger/DSPRegisterView.cpp
	Debugger/DebuggerPanel.cpp
	Debugger/DebuggerUIUtil.cpp
	Debugger/JitWindow.cpp
	Debugger/MemoryCheckDlg.cpp
	Debugger/MemoryView.cpp
	Debugger/MemoryWindow.cpp
	Debugger/RegisterView.cpp
	Debugger/RegisterWindow.cpp
	Debugger/WatchView.cpp
	Debugger/WatchWindow.cpp
	FifoPlayerDlg.cpp
	Frame.cpp
	FrameAui.cpp
	FrameTools.cpp
	GameListCtrl.cpp
	HotkeyDlg.cpp
	ISOFile.cpp
	ISOProperties.cpp
	InputConfigDiag.cpp
	InputConfigDiagBitmaps.cpp
	LogConfigWindow.cpp
	LogWindow.cpp
	Main.cpp
	MemcardManager.cpp
	NetWindow.cpp
	PatchAddEdit.cpp
	PostProcessingConfigDiag.cpp
	SoftwareVideoConfigDialog.cpp
	TASInputDlg.cpp
	VideoConfigDiag.cpp
	WXInputBase.cpp
	WiimoteConfigDiag.cpp
	WxUtils.cpp)

set(NOGUI_SRCS MainNoGUI.cpp)

if(USE_X11)
	set(GUI_SRCS ${GUI_SRCS} X11Utils.cpp)
	set(NOGUI_SRCS ${NOGUI_SRCS} X11Utils.cpp)
endif()

set(WXLIBS ${wxWidgets_LIBRARIES} dl)

list(APPEND LIBS core uicommon)

if(ANDROID)
	list(APPEND LIBS png)
endif()

set(ANDROID_SRCS Android/ButtonManager.cpp
	 MainAndroid.cpp)

if(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
	if(wxWidgets_FOUND)
		list(APPEND WXLIBS
			${APPSERV_LIBRARY}
			${COCOA_LIBRARY}
			)
	endif()
	# Add resource files to application bundle.
	set(RESOURCES resources/Dolphin.icns)
	list(APPEND SRCS ${RESOURCES})
	set_source_files_properties(${RESOURCES} PROPERTIES
		MACOSX_PACKAGE_LOCATION Resources)
endif()

if(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
	set(DOLPHIN_EXE_BASE Dolphin)
else()
	set(DOLPHIN_EXE_BASE dolphin-emu)
endif()

set(DOLPHIN_EXE ${DOLPHIN_EXE_BASE})

include(FindGettext)
if(GETTEXT_MSGMERGE_EXECUTABLE AND GETTEXT_MSGFMT_EXECUTABLE AND wxWidgets_FOUND)
	file(GLOB LINGUAS ${CMAKE_SOURCE_DIR}/Languages/po/*.po)
	add_custom_target(translations ALL)
	GETTEXT_CREATE_TRANSLATIONS(${CMAKE_SOURCE_DIR}/Languages/po/dolphin-emu.pot ${LINGUAS})
endif()

if(ANDROID)
	set(DOLPHIN_EXE main)
	add_library(${DOLPHIN_EXE} SHARED ${SRCS} ${ANDROID_SRCS})
	target_link_libraries(${DOLPHIN_EXE}
	log
	android
	"-Wl,--no-warn-mismatch"
	"-Wl,--whole-archive"
	${LIBS}
	"-Wl,--no-whole-archive"
	)
	add_custom_command(TARGET ${DOLPHIN_EXE} POST_BUILD
		COMMAND mkdir ARGS -p ${CMAKE_SOURCE_DIR}/Source/Android/libs/${ANDROID_NDK_OUTPUT_ABI_NAME}
	)
	add_custom_command(TARGET ${DOLPHIN_EXE} POST_BUILD
		COMMAND cp ARGS ${LIBRARY_OUTPUT_PATH_ROOT}/libs/${ANDROID_NDK_OUTPUT_ABI_NAME}/lib${DOLPHIN_EXE}.so ${CMAKE_SOURCE_DIR}/Source/Android/libs/${ANDROID_NDK_OUTPUT_ABI_NAME}/
	)
	add_custom_command(TARGET ${DOLPHIN_EXE} POST_BUILD
		COMMAND cp ARGS ${CMAKE_SOURCE_DIR}/Data/Sys/GC/* ${CMAKE_SOURCE_DIR}/Source/Android/assets/
	)
	add_custom_command(TARGET ${DOLPHIN_EXE} POST_BUILD
		COMMAND cp ARGS -r ${CMAKE_SOURCE_DIR}/Data/Sys/Shaders ${CMAKE_SOURCE_DIR}/Source/Android/assets/
	)

	set(CPACK_PACKAGE_EXECUTABLES ${CPACK_PACKAGE_EXECUTABLES} ${DOLPHIN_EXE})
elseif(wxWidgets_FOUND)
	add_executable(${DOLPHIN_EXE} ${SRCS} ${GUI_SRCS})
	target_link_libraries(${DOLPHIN_EXE} ${LIBS} ${WXLIBS})
	if(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
		include(BundleUtilities)
		set(BUNDLE_PATH ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/${DOLPHIN_EXE}.app)

		# Ask for an application bundle.
		set_target_properties(${DOLPHIN_EXE} PROPERTIES
			MACOSX_BUNDLE true
			MACOSX_BUNDLE_INFO_PLIST ${CMAKE_CURRENT_SOURCE_DIR}/Info.plist.in
			)

		# get rid of any old copies
		file (REMOVE_RECURSE ${BUNDLE_PATH}/Contents/Resources/Sys)
		if(NOT SKIP_POSTPROCESS_BUNDLE)
			# Fix up the bundle after it is finished.
			# There does not seem to be an easy way to run CMake commands post-build,
			# so we invoke CMake again on a generated script.
			file(WRITE ${CMAKE_CURRENT_BINARY_DIR}/postprocess_bundle.cmake "
				include(BundleUtilities)
				message(\"Fixing up application bundle: ${BUNDLE_PATH}\")
				message(\"(Note: This is only necessary to produce a redistributable binary.\")
				message(\"To skip, pass -DSKIP_POSTPROCESS_BUNDLE=1 to cmake.)\")
				set(BU_CHMOD_BUNDLE_ITEMS ON)
				fixup_bundle(\"${BUNDLE_PATH}\" \"\" \"\")
				file(INSTALL \"${CMAKE_SOURCE_DIR}/Data/Sys\"
					DESTINATION \"${BUNDLE_PATH}/Contents/Resources\"
					)
				")
			add_custom_command(TARGET ${DOLPHIN_EXE} POST_BUILD
				COMMAND ${CMAKE_COMMAND} -P postprocess_bundle.cmake
				)
		else()
			add_custom_command(OUTPUT ${BUNDLE_PATH}/Contents/Resources/Sys
				COMMAND ln -nfs ${CMAKE_SOURCE_DIR}/Data/Sys ${BUNDLE_PATH}/Contents/Resources/Sys
				VERBATIM
				)
			add_custom_target(CopyDataIntoBundleWx ALL
				DEPENDS ${BUNDLE_PATH}/Contents/Resources/Sys
				)
		endif()

		if(GETTEXT_MSGMERGE_EXECUTABLE AND GETTEXT_MSGFMT_EXECUTABLE AND wxWidgets_FOUND)
			file(WRITE ${CMAKE_CURRENT_BINARY_DIR}/copy_translations_into_bundle.cmake "
				file(GLOB TRANSLATION_FILES RELATIVE ${CMAKE_CURRENT_BINARY_DIR}
					${CMAKE_CURRENT_BINARY_DIR}/*.gmo
					)
				foreach(TRANSLATION_FILE \${TRANSLATION_FILES})
					string(REPLACE \".gmo\" \".lproj\" TRANSLATION_DIR
						\${TRANSLATION_FILE}
						)
					# It would be better to copy to the new name as a single action,
					# but I can't figure out a way to let CMake do that.
					file(COPY ${CMAKE_CURRENT_BINARY_DIR}/\${TRANSLATION_FILE}
						DESTINATION ${BUNDLE_PATH}/Contents/Resources/\${TRANSLATION_DIR}
						NO_SOURCE_PERMISSIONS
						)
					file(RENAME
						${BUNDLE_PATH}/Contents/Resources/\${TRANSLATION_DIR}/\${TRANSLATION_FILE}
						${BUNDLE_PATH}/Contents/Resources/\${TRANSLATION_DIR}/dolphin-emu.mo
						)
				endforeach(TRANSLATION_FILE)
				")

			file(GLOB PO_FILES RELATIVE ${CMAKE_SOURCE_DIR}/Languages/po
				${CMAKE_SOURCE_DIR}/Languages/po/*.po
				)
			string(REPLACE .po .gmo GMO_FILES "${PO_FILES}")

			add_custom_command(OUTPUT ${BUNDLE_PATH}/Contents/Resources/en.lproj
				COMMAND ${CMAKE_COMMAND} -P copy_translations_into_bundle.cmake
				DEPENDS ${GMO_FILES}
				${CMAKE_SOURCE_DIR}/Data/Sys
				VERBATIM
				)
			add_custom_target(CopyTranslationsIntoBundleWx ALL
				DEPENDS ${BUNDLE_PATH}/Contents/Resources/en.lproj
				)
		endif()

		# Install bundle into systemwide /Applications directory.
		install(DIRECTORY ${BUNDLE_PATH} DESTINATION /Applications
			USE_SOURCE_PERMISSIONS
			)
	else()
		install(TARGETS ${DOLPHIN_EXE} RUNTIME DESTINATION ${bindir})
	endif()

	set(CPACK_PACKAGE_EXECUTABLES ${CPACK_PACKAGE_EXECUTABLES} ${DOLPHIN_EXE})
endif()

if(USE_X11)
	set(DOLPHIN_NOGUI_EXE ${DOLPHIN_EXE_BASE}-nogui)
	add_executable(${DOLPHIN_NOGUI_EXE} ${SRCS} ${NOGUI_SRCS})
	target_link_libraries(${DOLPHIN_NOGUI_EXE} ${LIBS})
	set(CPACK_PACKAGE_EXECUTABLES ${CPACK_PACKAGE_EXECUTABLES} ${DOLPHIN_NOGUI_EXE})
	install(TARGETS ${DOLPHIN_NOGUI_EXE} RUNTIME DESTINATION ${bindir})
endif()
