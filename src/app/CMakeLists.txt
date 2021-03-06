add_executable(qBittorrent
# headers
application.h
applicationinstancemanager.h
cmdoptions.h
filelogger.h
qtlocalpeer/qtlocalpeer.h
upgrade.h

# sources
application.cpp
applicationinstancemanager.cpp
cmdoptions.cpp
filelogger.cpp
main.cpp
qtlocalpeer/qtlocalpeer.cpp
upgrade.cpp
)

target_include_directories(qBittorrent PRIVATE ${CMAKE_CURRENT_BINARY_DIR})
target_link_libraries(qBittorrent
    PRIVATE
        qbt_base
)

set_target_properties(qBittorrent
    PROPERTIES
        AUTOUIC True
        AUTORCC True
        MACOSX_BUNDLE True
)

# translations
include(QbtTranslations)

file(GLOB QBT_TS_FILES ../lang/*.ts)
qbt_add_translations(qBittorrent QRC_FILE "../lang/lang.qrc" TS_FILES ${QBT_TS_FILES})

if (NOT DISABLE_WEBUI)
    file(GLOB QBT_WEBUI_TS_FILES ../webui/www/translations/*.ts)
    qbt_add_translations(qBittorrent QRC_FILE "../webui/www/translations/webui_translations.qrc" TS_FILES ${QBT_WEBUI_TS_FILES})
endif()

set(QBT_APP_RESOURCES
../icons/icons.qrc
../searchengine/searchengine.qrc
)

# With AUTORCC rcc is ran by cmake before language files are generated,
# and thus we call rcc explicitly
qt5_add_resources(QBT_APP_RESOURCE_SOURCE ${QBT_APP_RESOURCES})

if (WIN32)
    if (MINGW)
        target_sources(qBittorrent PRIVATE ../qbittorrent_mingw.rc)
    else (MINGW)
        target_sources(qBittorrent PRIVATE ../qbittorrent.rc)
    endif (MINGW)
    target_sources(qBittorrent PRIVATE ../qbittorrent.exe.manifest)
endif (WIN32)

if (STACKTRACE)
    if (UNIX)
        target_sources(qBittorrent PRIVATE stacktrace.h)
    else (UNIX)
        target_sources(qBittorrent PRIVATE stacktrace_win.h)
        if (Qt5Widgets_FOUND)
            target_sources(qBittorrent PRIVATE stacktracedialog.cpp stacktracedialog.h)
        endif (Qt5Widgets_FOUND)
        if (MSVC)
            if (NOT "${WINXXBITS}" STREQUAL "Win64")
                # i686 arch requires frame pointer preservation
                add_compile_options(-Oy-)
            endif (NOT "${WINXXBITS}" STREQUAL "Win64")
            add_compile_options(-Zi)
            target_link_libraries(qBittorrent PUBLIC dbghelp -DEBUG)
        else (MSVC)
            if (NOT "${WINXXBITS}" STREQUAL "Win64")
                add_compile_options(-fno-omit-frame-pointer)
            endif (NOT "${WINXXBITS}" STREQUAL "Win64")
            target_link_libraries(qBittorrent PUBLIC dbghelp -Wl,--export-all-symbols)
        endif (MSVC)
    endif (UNIX)
endif (STACKTRACE)

if (Qt5Widgets_FOUND)
    target_link_libraries(qBittorrent PRIVATE qbt_searchengine qbt_gui)
    set_target_properties(qBittorrent
        PROPERTIES
            OUTPUT_NAME qbittorrent
            WIN32_EXECUTABLE True
    )
else(Qt5Widgets_FOUND)
    set_target_properties(qBittorrent
        PROPERTIES
            OUTPUT_NAME qbittorrent-nox
    )
endif (Qt5Widgets_FOUND)

if (NOT DISABLE_WEBUI)
    target_link_libraries(qBittorrent PRIVATE qbt_webui)
endif()

# we have to include resources into the bundle
if (APPLE)
    set(OSX_RES_SRC_DIR "${qBittorrent_SOURCE_DIR}/dist/mac")
    list(APPEND QBT_APP_RESOURCE_SOURCE
        "${OSX_RES_SRC_DIR}/qt.conf"
        "${OSX_RES_SRC_DIR}/qBitTorrentDocument.icns"
        "${OSX_RES_SRC_DIR}/qbittorrent_mac.icns")
    set_source_files_properties(
        "${OSX_RES_SRC_DIR}/qt.conf"
        "${OSX_RES_SRC_DIR}/qBitTorrentDocument.icns"
        "${OSX_RES_SRC_DIR}/qbittorrent_mac.icns"
        PROPERTIES
            MACOSX_PACKAGE_LOCATION Resources)
    set(QT_TR_DIR "${qBittorrent_SOURCE_DIR}/dist/qt-translations")
    set(QT_TRANSLATIONS
        ${QT_TR_DIR}/qt_ar.qm
        ${QT_TR_DIR}/qt_bg.qm
        ${QT_TR_DIR}/qt_ca.qm
        ${QT_TR_DIR}/qt_cs.qm
        ${QT_TR_DIR}/qt_da.qm
        ${QT_TR_DIR}/qt_de.qm
        ${QT_TR_DIR}/qt_es.qm
        ${QT_TR_DIR}/qt_eu.qm
        ${QT_TR_DIR}/qt_fi.qm
        ${QT_TR_DIR}/qt_fr.qm
        ${QT_TR_DIR}/qt_gl.qm
        ${QT_TR_DIR}/qt_he.qm
        ${QT_TR_DIR}/qt_hu.qm
        ${QT_TR_DIR}/qt_it.qm
        ${QT_TR_DIR}/qt_ja.qm
        ${QT_TR_DIR}/qt_ko.qm
        ${QT_TR_DIR}/qt_lt.qm
        ${QT_TR_DIR}/qt_nl.qm
        ${QT_TR_DIR}/qt_pl.qm
        ${QT_TR_DIR}/qt_pt.qm
        ${QT_TR_DIR}/qt_pt_BR.qm
        ${QT_TR_DIR}/qt_ru.qm
        ${QT_TR_DIR}/qt_sk.qm
        ${QT_TR_DIR}/qt_sv.qm
        ${QT_TR_DIR}/qt_tr.qm
        ${QT_TR_DIR}/qt_uk.qm
        ${QT_TR_DIR}/qt_zh_CN.qm
        ${QT_TR_DIR}/qt_zh_TW.qm
    )
    list(APPEND QBT_APP_RESOURCE_SOURCE ${QT_TRANSLATIONS})
    set_source_files_properties(${QT_TRANSLATIONS}
        PROPERTIES MACOSX_PACKAGE_LOCATION translations)
endif (APPLE)

target_sources(qBittorrent PRIVATE ${QBT_QM_FILES} ${QBT_APP_RESOURCE_SOURCE})

get_target_property(QBT_EXECUTABLE_NAME qBittorrent OUTPUT_NAME)

if (APPLE)
    set(qbt_BUNDLE_NAME ${QBT_EXECUTABLE_NAME})

# substitute @EXECUTABLE@ in dist/mac/Info.plist
    set(EXECUTABLE ${qbt_BUNDLE_NAME})
    configure_file(${qBittorrent_SOURCE_DIR}/dist/mac/Info.plist ${qBittorrent_BINARY_DIR}/dist/mac/Info.plist @ONLY)

    set_target_properties(qBittorrent PROPERTIES
        MACOSX_BUNDLE_BUNDLE_NAME "${qbt_BUNDLE_NAME}"
        MACOSX_BUNDLE_INFO_PLIST ${qBittorrent_BINARY_DIR}/dist/mac/Info.plist
    )
endif (APPLE)

# installation
install(TARGETS qBittorrent
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
    BUNDLE  DESTINATION .
    COMPONENT runtime)

if (Qt5Widgets_FOUND AND APPLE)
    find_package(Qt5Svg REQUIRED)
    include(bundle)
endif (Qt5Widgets_FOUND AND APPLE)
