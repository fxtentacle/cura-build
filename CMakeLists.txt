project(cura-build)

cmake_minimum_required(VERSION 2.8.12)

set(CMAKE_MODULE_PATH ${CMAKE_SOURCE_DIR}/cmake)

include(ExternalProject)
include(GNUInstallDirs)

if(APPLE)
    set(CMAKE_OSX_DEPLOYMENT_TARGET "10.7")
endif()

option(BUILD_PYTHON "Include Python" ON)
option(BUILD_QT "Include Qt" ON)
option(BUILD_PYQT "Include PyQt" ON)
option(BUILD_NUMPY "Include Numpy" ON)
option(BUILD_PYSERIAL "Include PySerial" ON)

if(WIN32)
    option(BUILD_64BIT "Create a 64-bit build" OFF)
endif()

if(WIN32)
    if (BUILD_64BIT)
        set(BUILD_PYTHON OFF)
        set(BUILD_QT OFF)
        set(BUILD_PYQT OFF) #PyQt Fails to properly build with MinGW 64-bit
        set(BUILD_NUMPY OFF) #Numpy also fails
    else()
        set(BUILD_PYTHON OFF)
    endif()
elseif(NOT APPLE)
    set(BUILD_PYTHON OFF)
    set(BUILD_QT OFF)
    set(BUILD_PYQT OFF)
    set(BUILD_NUMPY OFF)
    set(BUILD_PYSERIAL OFF)
endif()

set(CURA_MAJOR_VERSION "00")
set(CURA_MINOR_VERSION "00")
set(CURA_PATCH_VERSION "00")

set(TAG_OR_BRANCH "15.10" CACHE STRING "The name of the tag or branch to build")
set(EXTERNALPROJECT_INSTALL_PREFIX ${CMAKE_BINARY_DIR}/inst)
set(EXTERNALPROJECT_SITEPACKAGES_DIR ${EXTERNALPROJECT_INSTALL_PREFIX}/lib/python3.4/site-packages)

if(BUILD_PYTHON)
    ExternalProject_Add(Python
        URL https://www.python.org/ftp/python/3.4.3/Python-3.4.3.tgz
        CONFIGURE_COMMAND ./configure --prefix=${EXTERNALPROJECT_INSTALL_PREFIX} --enable-shared --with-threads --without-pymalloc
        BUILD_IN_SOURCE 1
    )

    set(PYTHON_EXECUTABLE ${EXTERNALPROJECT_INSTALL_PREFIX}/bin/python3)
    set(PYTHON_VERSION_MAJOR 3)
    set(PYTHON_VERSION_MINOR 4)
else()
    find_package(PythonInterp 3.4.0 REQUIRED)
    add_custom_target(Python)
endif()

if(BUILD_QT)
    set(qt_url http://download.qt.io/archive/qt/5.4/5.4.2/single/qt-everywhere-opensource-src-5.4.2.tar.gz)
    set(qt_md5 fa1c4d819b401b267eb246a543a63ea5)
    if(WIN32)
        set(qt_url http://download.qt.io/archive/qt/5.4/5.4.2/single/qt-everywhere-opensource-src-5.4.2.zip)
        set(qt_md5 b6a099fff154fcb0a0127619623d412f)
    endif()

    set(qt_options
        -release
        -prefix ${EXTERNALPROJECT_INSTALL_PREFIX}
        -archdatadir ${EXTERNALPROJECT_INSTALL_PREFIX}/lib
        -datadir ${EXTERNALPROJECT_INSTALL_PREFIX}/share
        -opensource
        -confirm-license
        -nomake examples
        -nomake tests
        -skip qtconnectivity
        -skip qtdoc
        -skip qtenginio
        -skip qtlocation
        -skip qtmultimedia
        -skip qtquick1
        -skip qtscript
        -skip qtsensors
        -skip qtwebchannel
        -skip qtwebengine
        -skip qtwebkit
        -skip qtwebsockets
        -skip qtx11extras
        -skip qtandroidextras
        -skip qtactiveqt
        -skip qttools
        -skip qtxmlpatterns
    )

    if(APPLE)
        list(APPEND qt_options -no-framework)
    elseif(WIN32)
        list(APPEND qt_options -opengl desktop)
    endif()

    ExternalProject_Add(Qt
        URL ${qt_url}
        URL_MD5 ${qt_md5}
        CONFIGURE_COMMAND ./configure ${qt_options}
        BUILD_IN_SOURCE 1
    )
else()
    add_custom_target(Qt)
endif()

if(BUILD_PYQT)
    set(sip_command "")
    if(WIN32)
        set(sip_command
            ${PYTHON_EXECUTABLE}
            configure.py
            --platform win32-g++
            --bindir=${EXTERNALPROJECT_INSTALL_PREFIX}/bin
            --destdir=${EXTERNALPROJECT_INSTALL_PREFIX}/lib/python3.4/site-packages
            --incdir=${EXTERNALPROJECT_INSTALL_PREFIX}/include
            --sipdir=${EXTERNALPROJECT_INSTALL_PREFIX}/share/sip
            "CFLAGS+=${CMAKE_C_FLAGS}"
            "CXXFLAGS+=${CMAKE_CXX_FLAGS}"
        )
    else()
        set(sip_command ${PYTHON_EXECUTABLE} configure.py --sysroot=${EXTERNALPROJECT_INSTALL_PREFIX})
    endif()

    ExternalProject_Add(Sip
        DEPENDS Python
        URL http://downloads.sourceforge.net/project/pyqt/sip/sip-4.16.9/sip-4.16.9.zip
        CONFIGURE_COMMAND ${sip_command}
        BUILD_IN_SOURCE 1
    )

    set(pyqt_command "")
    if(WIN32)
        set(pyqt_command
            ${PYTHON_EXECUTABLE} configure.py
            --sip ${EXTERNALPROJECT_INSTALL_PREFIX}/bin/sip.exe
            --confirm-license
            --spec win32-g++
            --destdir ${EXTERNALPROJECT_INSTALL_PREFIX}/lib/python3.4/site-packages
            --bindir ${EXTERNALPROJECT_INSTALL_PREFIX}/bin
            --sipdir ${EXTERNALPROJECT_INSTALL_PREFIX}/share/sip
            "CFLAGS+=${CMAKE_C_FLAGS}"
            "CXXFLAGS+=${CMAKE_CXX_FLAGS}"
        )
    else()
        set(pyqt_command
             ${PYTHON_EXECUTABLE} configure.py
            --sysroot ${EXTERNALPROJECT_INSTALL_PREFIX}
            --qmake ${EXTERNALPROJECT_INSTALL_PREFIX}/bin/qmake
            --sip ${EXTERNALPROJECT_INSTALL_PREFIX}/bin/sip
            --confirm-license
        )
    endif()

    ExternalProject_Add(PyQt
        DEPENDS Python Qt Sip
        URL http://downloads.sourceforge.net/project/pyqt/PyQt5/PyQt-5.4.2/PyQt-gpl-5.4.2.tar.gz
        CONFIGURE_COMMAND ${pyqt_command}
        BUILD_IN_SOURCE 1
    )
else()
    add_custom_target(PyQt)
endif()

if(WIN32)
    # Because setuptools is stupid and will interpret "C:/..." as a relative path
    string(REPLACE "/" "\\" native_prefix ${EXTERNALPROJECT_INSTALL_PREFIX})
    set(python_install
        --install-lib=${native_prefix}\\lib\\python3.4\\site-packages 
        --install-data=${native_prefix}\\share
        --install-scripts=${native_prefix}\\bin
    )
endif()

if(BUILD_NUMPY)
    ExternalProject_Add(Numpy
        DEPENDS Python
        URL http://downloads.sourceforge.net/project/numpy/NumPy/1.9.2/numpy-1.9.2.tar.gz
        URL_MD5 a1ed53432dbcd256398898d35bc8e645
        CONFIGURE_COMMAND ""
        BUILD_COMMAND ${PYTHON_EXECUTABLE} setup.py build
        INSTALL_COMMAND ${PYTHON_EXECUTABLE} setup.py install ${python_install}
        BUILD_IN_SOURCE 1
    )
endif()

if(BUILD_PYSERIAL)
    ExternalProject_Add(PySerial
        DEPENDS Python
        URL https://pypi.python.org/packages/source/p/pyserial/pyserial-2.7.tar.gz
        URL_MD5 794506184df83ef2290de0d18803dd11
        CONFIGURE_COMMAND ""
        BUILD_COMMAND ${PYTHON_EXECUTABLE} setup.py build
        INSTALL_COMMAND ${PYTHON_EXECUTABLE} setup.py install ${python_install}
        BUILD_IN_SOURCE 1
    )
endif()

ExternalProject_Add(Protobuf 
    DEPENDS Python
    GIT_REPOSITORY https://github.com/ultimaker/protobuf
    GIT_TAG ${TAG_OR_BRANCH}
    CMAKE_ARGS -DCMAKE_INSTALL_PREFIX=${EXTERNALPROJECT_INSTALL_PREFIX} -DCMAKE_BUILD_TYPE=Release -DINSTALL_PYTHON=ON -DBUILD_STATIC=ON
)

ExternalProject_Add(Arcus 
    DEPENDS Python Protobuf
    GIT_REPOSITORY https://github.com/ultimaker/libArcus
    GIT_TAG ${TAG_OR_BRANCH}
    CMAKE_ARGS -DCMAKE_INSTALL_PREFIX=${EXTERNALPROJECT_INSTALL_PREFIX} -DCMAKE_BUILD_TYPE=Release -DBUILD_EXAMPLES=OFF -DBUILD_STATIC=ON
)

ExternalProject_Add(CuraEngine
    DEPENDS Arcus
    GIT_REPOSITORY https://github.com/ultimaker/CuraEngine
    GIT_TAG ${TAG_OR_BRANCH}
    CMAKE_ARGS -DCMAKE_INSTALL_PREFIX=${EXTERNALPROJECT_INSTALL_PREFIX} -DCMAKE_BUILD_TYPE=Release
)

ExternalProject_Add(Uranium
    DEPENDS PyQt Arcus
    GIT_REPOSITORY https://github.com/ultimaker/Uranium
    GIT_TAG ${TAG_OR_BRANCH}
    CMAKE_ARGS -DCMAKE_INSTALL_PREFIX=${EXTERNALPROJECT_INSTALL_PREFIX}
)

ExternalProject_Add(Cura
    DEPENDS Uranium CuraEngine
    GIT_REPOSITORY https://github.com/ultimaker/Cura
    GIT_TAG ${TAG_OR_BRANCH}
    CMAKE_ARGS -DCMAKE_INSTALL_PREFIX=${EXTERNALPROJECT_INSTALL_PREFIX} -DURANIUM_SCRIPTS_DIR=
)

ExternalProject_Add(Cura-PostProcessing
    DEPENDS Cura
    GIT_REPOSITORY https://github.com/nallath/PostProcessingPlugin
    CMAKE_ARGS -DCMAKE_INSTALL_PREFIX=${EXTERNALPROJECT_INSTALL_PREFIX}
)

if(APPLE)
    ExternalProject_Add(Py2App
        DEPENDS Python
        URL https://pypi.python.org/packages/source/p/py2app/py2app-0.9.tar.gz
        CONFIGURE_COMMAND ""
        BUILD_COMMAND ${PYTHON_EXECUTABLE} setup.py build
        INSTALL_COMMAND ${PYTHON_EXECUTABLE} setup.py install
        BUILD_IN_SOURCE 1
    )
elseif(WIN32)
    # Currently, py2exe can not be built with MinGW since the code apparently uses
    # a lot of MSVC-isms. So instead, install py2exe in your Python installation and
    # use that.
    # ExternalProject_Add(Py2Exe
        # DEPENDS Python
        # URL https://pypi.python.org/packages/source/p/py2exe/py2exe-0.9.2.0.zip
        # CONFIGURE_COMMAND ""
        # BUILD_COMMAND ${PYTHON_EXECUTABLE} setup.py build
        # INSTALL_COMMAND ${PYTHON_EXECUTABLE} setup.py install ${python_install}
        # BUILD_IN_SOURCE 1
    # )
endif()

if(APPLE)
    configure_file(setup_osx.py.in setup_osx.py @ONLY)

    add_custom_command(
        OUTPUT ${CMAKE_BINARY_DIR}/dist/Cura.app 
        COMMAND ${PYTHON_EXECUTABLE}
        ARGS setup_osx.py py2app
        COMMENT "Running py2app"
        WORKING_DIRECTORY ${CMAKE_BINARY_DIR}
    )

    add_custom_target(build_app ALL DEPENDS ${CMAKE_BINARY_DIR}/dist/Cura.app)
    install(DIRECTORY ${CMAKE_BINARY_DIR}/dist/Cura.app DESTINATION "." USE_SOURCE_PERMISSIONS)

    set(CPACK_GENERATOR "DragNDrop")
elseif(WIN32)
    get_filename_component(compiler_dir ${CMAKE_CXX_COMPILER} DIRECTORY)
    configure_file(setup_win32.py.in setup_win32.py @ONLY)

    add_custom_command(
        OUTPUT ${CMAKE_BINARY_DIR}/dist/Cura.exe
        COMMAND ${PYTHON_EXECUTABLE}
        ARGS setup_win32.py py2exe
        COMMENT "Running py2exe"
        WORKING_DIRECTORY ${CMAKE_BINARY_DIR}
    )

    add_custom_target(build_exe ALL DEPENDS ${CMAKE_BINARY_DIR}/dist/Cura.exe)
    install(DIRECTORY ${CMAKE_BINARY_DIR}/dist/ DESTINATION "." USE_SOURCE_PERMISSIONS COMPONENT "cura")
    # Get the VCRedist from http://www.microsoft.com/en-us/download/details.aspx?id=26999
    if(BUILD_64BIT)
        install(FILES vcredist_x64.exe DESTINATION "." COMPONENT "vcredist")
    else()
        install(FILES vcredist_x86.exe DESTINATION "." COMPONENT "vcredist")
    endif()
    install(DIRECTORY arduino DESTINATION "." COMPONENT "arduino")

    include(CPackComponent)
    cpack_add_component(cura DISPLAY_NAME "Cura Executable and Data Files" REQUIRED)
    cpack_add_component(vcredist DISPLAY_NAME "Install Visual Studio 2010 Redistributable")
    cpack_add_component(arduino DISPLAY_NAME "Install Arduino Drivers")

    set(CPACK_GENERATOR "NSIS")
else()
    set(CPACK_GENERATOR "DEB")

    set(CPACK_INSTALL_CMAKE_PROJECTS
        "${CMAKE_BINARY_DIR}/Arcus-prefix/src/Arcus-build;Arcus;ALL;/usr"
        "${CMAKE_BINARY_DIR}/CuraEngine-prefix/src/CuraEngine-build;CuraEngine;ALL;/usr"
        "${CMAKE_BINARY_DIR}/Uranium-prefix/src/Uranium-build;Uranium;ALL;/usr"
        "${CMAKE_BINARY_DIR}/Cura-prefix/src/Cura-build;Cura;ALL;/usr"
        "${CMAKE_BINARY_DIR}/Cura-PostProcessing-prefix/src/Cura-PostProcessing-build;Cura-PostProcessing;ALL;/usr"
    )
    set(CPACK_INSTALLED_DIRECTORIES "${CMAKE_BINARY_DIR}/inst/lib/python3.4/site-packages/google;/usr/lib/python3/dist-packages/google")
endif()

set(CPACK_PACKAGE_NAME "Cura")
set(CPACK_PACKAGE_VENDOR "Ultimaker")
set(CPACK_PACKAGE_VERSION_MAJOR ${CURA_MAJOR_VERSION})
set(CPACK_PACKAGE_VERSION_MINOR ${CURA_MINOR_VERSION})
set(CPACK_PACKAGE_VERSION_PATCH ${CURA_PATCH_VERSION})
set(CPACK_PACKAGE_DESCRIPTION_SUMMARY "Cura 3D Printing Software")
set(CPACK_PACKAGE_CONTACT "Arjen Hiemstra <a.hiemstra@ultimaker.com>")
set(CPACK_RESOURCE_FILE_LICENSE ${CMAKE_SOURCE_DIR}/LICENSE)
set(CPACK_PACKAGE_EXECUTABLES Cura "Cura ${CURA_MAJOR_VERSION}.${CURA_MINOR_VERSION}.${CURA_PATCH_VERSION}")
set(CPACK_PACKAGE_INSTALL_DIRECTORY "Cura ${CURA_MAJOR_VERSION}.${CURA_MINOR_VERSION}")

set(RPM_REQUIRES
    "python3 >= 3.4.0"
    "python3-qt5 >= 5.4.0"
    "python3-numpy >= 1.9.0"
    "qt5-qtquickcontrols >= 5.4.0"
)
string(REPLACE ";" "," RPM_REQUIRES "${RPM_REQUIRES}")
set(CPACK_RPM_PACKAGE_REQUIRES ${RPM_REQUIRES})
set(CPACK_RPM_PACKAGE_RELOCATABLE OFF)

set(DEB_DEPENDS
    "python3 (>= 3.4.0)"
    "python3-pyqt5 (>= 5.4.0)"
    "python3-pyqt5.qtopengl (>= 5.4.0)"
    "python3-pyqt5.qtquick (>= 5.4.0)"
    "python3-pyqt5.qtsvg (>= 5.4.0)"
    "python3-numpy (>= 1.8.0)"
    "python3-serial (>= 2.6)"
    "python3-opengl (>= 3.0)"
    "qml-module-qtquick2 (>= 5.4.0)"
    "qml-module-qtquick-window2 (>= 5.4.0)"
    "qml-module-qtquick-layouts (>= 5.4.0)"
    "qml-module-qtquick-dialogs (>= 5.4.0)"
    "qml-module-qtquick-controls (>= 5.4.0)"
)
string(REPLACE ";" "," DEB_DEPENDS "${DEB_DEPENDS}")
set(CPACK_DEBIAN_PACKAGE_DEPENDS ${DEB_DEPENDS})
set(CPACK_DEBIAN_PACKAGE_ARCHITECTURE amd64)

set(CPACK_NSIS_ENABLE_UNINSTALL_BEFORE_INSTALL ON)
set(CPACK_NSIS_EXECUTABLES_DIRECTORY ".")
set(CPACK_NSIS_MUI_FINISHPAGE_RUN "Cura.exe")
set(CPACK_NSIS_MENU_LINKS
    "https://ultimaker.com/en/support/software" "Cura Online Documentation"
    "https://github.com/ultimaker/cura" "Cura Development Resources"
)

include(CPack)
