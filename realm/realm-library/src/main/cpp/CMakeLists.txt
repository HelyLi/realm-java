cmake_minimum_required(VERSION 3.6.0)

# find javah
find_package(Java COMPONENTS Development)
if (NOT Java_Development_FOUND)
    if (DEFINED ENV{JAVA_HOME} AND EXISTS "$ENV{JAVA_HOME}/bin/javah")
      set(Java_JAVAH_EXECUTABLE "$ENV{JAVA_HOME}/bin/javah")
    elseif (EXISTS "/usr/bin/javah")
      set(Java_JAVAH_EXECUTABLE "/usr/bin/javah")
    else()
      message(FATAL_ERROR "Cannot find javah")
    endif()
endif()
include (UseJava)

set(CMAKE_VERBOSE_MAKEFILE ON)
# Generate compile_commands.json
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

# Setup lcache
if(NDK_LCACHE)
    set(CMAKE_CXX_CREATE_SHARED_LIBRARY "${NDK_LCACHE} ${CMAKE_CXX_CREATE_SHARED_LIBRARY}")
endif()

# Set flag build_SYNC
if (REALM_FLAVOR STREQUAL base)
    set(build_SYNC OFF)
else()
    set(build_SYNC ON)
endif()

# Generate JNI header files. Each build has its own JNI header in its build_dir/jni_include.
string(TOLOWER ${CMAKE_BUILD_TYPE} build_type_FOLDER)
set(classes_PATH ${CMAKE_SOURCE_DIR}/../../../build/intermediates/classes/${REALM_FLAVOR}/${build_type_FOLDER}/)
set(classes_LIST
    io.realm.internal.Table io.realm.internal.TableView io.realm.internal.CheckedRow
    io.realm.internal.LinkView io.realm.internal.Util io.realm.internal.UncheckedRow
    io.realm.internal.TableQuery io.realm.internal.SharedRealm io.realm.internal.TestUtil
    io.realm.log.LogLevel io.realm.Property io.realm.RealmSchema io.realm.RealmObjectSchema
)
set(jni_headers_PATH ${PROJECT_BINARY_DIR}/jni_include)
if (build_SYNC)
    list(APPEND classes_LIST
        io.realm.SyncManager io.realm.internal.objectserver.ObjectServerSession)
endif()
create_javah(TARGET jni_headers
    CLASSES ${classes_LIST}

    CLASSPATH ${classes_PATH}
    OUTPUT_DIR ${jni_headers_PATH}
    DEPENDS ${classes_PATH}
)

# TODO: Ideally the debug build should link with core's debug build. But core dbg lib has
# some compile options problems with arm, especially with macro REALM_DEBUG. Link to core
# dbg for debug build when that gets solved.
# We always link to the non-dbg version of core libs for now.
# This means only JNI part has debugging symbols with debug build.
# Debugging with core source code will also be done though anther approach -- compiling the core
# with cmake inside android project.
# Configure import realm core lib
set(core_lib_PATH ${REALM_CORE_DIST_DIR}/librealm-android-${ANDROID_ABI}.a)
# Workaround for old core's funny ABI nicknames
if (NOT EXISTS ${core_lib_PATH})
    if (ARMEABI)
        set(core_lib_PATH ${REALM_CORE_DIST_DIR}/librealm-android-arm.a)
    elseif (ARMEABI_V7A)
        set(core_lib_PATH ${REALM_CORE_DIST_DIR}/librealm-android-arm-v7a.a)
    elseif (ARM64_V8A)
        set(core_lib_PATH ${REALM_CORE_DIST_DIR}/librealm-android-arm64.a)
    else()
        message(FATAL_ERROR "Cannot find core lib file: ${core_lib_PATH}")
    endif()
endif()

add_library(lib_realm_core STATIC IMPORTED)
set_target_properties(lib_realm_core PROPERTIES IMPORTED_LOCATION ${core_lib_PATH})

# Sync static library
set(sync_lib_PATH ${REALM_CORE_DIST_DIR}/librealm-sync-android-${ANDROID_ABI}.a)
# Workaround for old core's funny ABI nicknames
if (NOT EXISTS ${sync_lib_PATH})
    if (ARMEABI)
        set(sync_lib_PATH ${REALM_CORE_DIST_DIR}/librealm-sync-android-arm.a)
    elseif (ARMEABI_V7A)
        set(sync_lib_PATH ${REALM_CORE_DIST_DIR}/librealm-sync-android-arm-v7a.a)
    elseif (ARM64_V8A)
        set(sync_lib_PATH ${REALM_CORE_DIST_DIR}/librealm-sync-android-arm64.a)
    else()
        message(FATAL_ERROR "Cannot find core lib file: ${core_lib_PATH}")
    endif()
endif()
add_library(lib_realm_sync STATIC IMPORTED)
set_target_properties(lib_realm_sync PROPERTIES IMPORTED_LOCATION ${sync_lib_PATH})

# build application's shared lib
include_directories(${REALM_CORE_DIST_DIR}/include
    ${CMAKE_SOURCE_DIR}
    ${jni_headers_PATH}
    ${CMAKE_SOURCE_DIR}/object-store/src)

set(ANDROID_STL "gnustl_static")
set(ANDROID_NO_UNDEFINED OFF)
set(ANDROID_SO_UNDEFINED ON)

if (ARMEABI)
    set(ABI_CXX_FLAGS "-mthumb")
elseif (ARMEABI_V7A)
    set(ABI_CXX_FLAGS "-mthumb -march=armv7-a -mfloat-abi=softfp -mfpu=vfpv3-d16")
endif()

#FIXME uninitialized is reported by query_expression.hpp:1070
#      d.init(ValueBase::m_from_link_list, ValueBase::m_values, D{});
#FIXME maybe-uninitialized is reported by table_view.cpp:272:15:
#     'best.m_nanoseconds' was declared here
set(WARNING_CXX_FLAGS "-Wall -Wextra -pedantic -Wno-long-long -Wno-variadic-macros \
-Wno-missing-field-initializers -Wmissing-declarations -Wno-error=uninitialized -Wno-error=maybe-uninitialized")
set(REALM_COMMON_CXX_FLAGS "-DREALM_ANDROID -DREALM_HAVE_CONFIG -DPIC -pthread -fvisibility=hidden -std=c++14 -fsigned-char")
if (build_SYNC)
    set(REALM_COMMON_CXX_FLAGS "${REALM_COMMON_CXX_FLAGS} -DREALM_SYNC")
endif()
# There might be an issue with -Os of ndk gcc 4.9. It will hang the encryption related tests.
# And this issue doesn't seem to impact the core compiling.
set(CMAKE_CXX_FLAGS_RELEASE "-O2 -DNDEBUG")
#-ggdb doesn't play well with -flto
set(CMAKE_CXX_FLAGS_DEBUG "-ggdb -Og -DNDEBUG")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${REALM_COMMON_CXX_FLAGS} ${WARNING_CXX_FLAGS} ${ABI_CXX_FLAGS}")

# Set link flags
set(REALM_LINKER_FLAGS "")
if (build_SYNC)
    set(REALM_LINKER_FLAGS "${REALM_LINKER_FLAGS} -lz")
endif()
set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} ${REALM_LINKER_FLAGS}")

# JNI source files
file(GLOB jni_SRC
    "*.cpp"
)
# Those source file are only needed for sync.
if (NOT build_SYNC)
    list(REMOVE_ITEM jni_SRC
        ${CMAKE_CURRENT_SOURCE_DIR}/io_realm_SyncManager.cpp
        ${CMAKE_CURRENT_SOURCE_DIR}/io_realm_internal_objectserver_ObjectServerSession.cpp)
endif()

# Object Store source files
file(GLOB objectstore_SRC
    "object-store/src/collection_notifications.cpp"
    "object-store/src/object_schema.cpp"
    "object-store/src/object_store.cpp"
    "object-store/src/schema.cpp"
    "object-store/src/index_set.cpp"
    "object-store/src/shared_realm.cpp"
    "object-store/src/impl/realm_coordinator.cpp"
    "object-store/src/impl/collection_notifier.cpp"
    "object-store/src/impl/collection_change_builder.cpp"
    "object-store/src/impl/transact_log_handler.cpp"
    "object-store/src/impl/weak_realm_notifier.cpp"
    "object-store/src/impl/android/*.cpp"
    "object-store/src/util/*.cpp")
# Sync needed Object Store files
if (build_SYNC)
    file(GLOB objectstore_sync_SRC
        "object-store/src/sync_manager.cpp"
        "object-store/src/sync_session.cpp")
endif()

add_library(realm-jni SHARED ${jni_SRC} ${objectstore_SRC} ${objectstore_sync_SRC})
add_dependencies(realm-jni jni_headers)
# -latomic is not set by default for mips. See https://code.google.com/p/android/issues/detail?id=182094
if (build_SYNC)
# FIXME: The order matters! lib_realm_sync needs to be in front of lib_realm_core!! Find out why!!
target_link_libraries(realm-jni log android atomic lib_realm_sync lib_realm_core)
else()
target_link_libraries(realm-jni log android atomic lib_realm_core)
endif()

# Strip the release so files and backup the unstripped versions
if (CMAKE_BUILD_TYPE STREQUAL "Release")
    set(unstripped_SO_DIR
        "${CMAKE_SOURCE_DIR}/../../../build/outputs/jniLibs-unstripped/${REALM_FLAVOR}/${ANDROID_ABI}")
    add_custom_command(TARGET realm-jni
        POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E make_directory ${unstripped_SO_DIR}
        COMMAND ${CMAKE_COMMAND} -E copy $<TARGET_FILE:realm-jni> ${unstripped_SO_DIR}
        COMMAND ${CMAKE_STRIP} $<TARGET_FILE:realm-jni>)
endif()
