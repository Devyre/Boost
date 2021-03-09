cmake_minimum_required(VERSION 3.18)

function(add_boost_lib)
    cmake_parse_arguments(
            boostlib
            ""
            "TARGET"
            ""
            ${ARGN}
    )

    file(GLOB_RECURSE src ${boost_SOURCE_DIR}/libs/${boostlib_TARGET}/src/*.cpp)

    add_library(Boost_${boostlib_TARGET} STATIC ${src})
    add_library(Boost::${boostlib_TARGET} ALIAS Boost_${boostlib_TARGET})
    set_target_properties(
            Boost_${boostlib_TARGET}
            PROPERTIES
            OUTPUT_NAME "boost_${boostlib_TARGET}"
            FOLDER "boost"
    )

    target_link_libraries(Boost_${boostlib_TARGET} PUBLIC Boost::boost)
    target_compile_definitions(
            Boost_${boostlib_TARGET}
            PUBLIC
            BOOST_NO_EXCEPTIONS=1
    )

    if (CMAKE_CXX_COMPILER_ID STREQUAL "MSVC" OR CMAKE_CXX_COMPILER_FRONTEND_VARIANT STREQUAL "MSVC")
        target_compile_options(Boost_${boostlib_TARGET} PRIVATE /W0)
    else()
        target_compile_options(Boost_${boostlib_TARGET} PRIVATE -w)
    endif()
endfunction()

if (NOT DEFINED OCTANE_EXTERNAL_DIR)
    set(OCTANE_EXTERNAL_DIR $ENV{OCTANE_EXTERNAL_DIR})

    if (NOT DEFINED OCTANE_EXTERNAL_DIR)
        set(OCTANE_EXTERNAL_DIR "${CMAKE_CURRENT_BINARY_DIR}/external")
    endif()
endif()

if (NOT DEFINED OCTANE_BOOST_VERSION)
    set(OCTANE_BOOST_VERSION $ENV{OCTANE_BOOST_VERSION})

    if (NOT DEFINED OCTANE_BOOST_VERSION)
        set(OCTANE_BOOST_VERSION "1.74.0")
    endif()
endif()

string(REPLACE "." "_" OCTANE_BOOST_FILE_VERSION ${OCTANE_BOOST_VERSION})

if (NOT DEFINED OCTANE_BOOST_URL)
    set(OCTANE_BOOST_URL $ENV{OCTANE_BOOST_URL})

    if (NOT DEFINED OCTANE_BOOST_URL)
        set(OCTANE_BOOST_URL "https://dl.bintray.com/boostorg/release/${OCTANE_BOOST_VERSION}/source/boost_${OCTANE_BOOST_FILE_VERSION}.tar.gz")
    endif()
endif()

message(STATUS "Boost target directory: ${OCTANE_EXTERNAL_DIR}/lib/boost_${OCTANE_BOOST_FILE_VERSION}")
message(STATUS "Configuring boost ${OCTANE_BOOST_VERSION}")

file(LOCK "${OCTANE_EXTERNAL_DIR}/lib/boost.lock")
set(boost_SOURCE_DIR "${OCTANE_EXTERNAL_DIR}/lib/boost_${OCTANE_BOOST_FILE_VERSION}")

if (NOT EXISTS "${OCTANE_EXTERNAL_DIR}/lib/boost_${OCTANE_BOOST_FILE_VERSION}.tar.gz")
    message(STATUS "Downloading boost ${OCTANE_BOOST_VERSION} from ${OCTANE_BOOST_URL}...")
    file(DOWNLOAD "${OCTANE_BOOST_URL}" "${OCTANE_EXTERNAL_DIR}/lib/boost_${OCTANE_BOOST_FILE_VERSION}.tar.gz")
endif()

if (NOT EXISTS "${OCTANE_EXTERNAL_DIR}/lib/boost_${OCTANE_BOOST_FILE_VERSION}")
    message(STATUS "Extracting boost_${OCTANE_BOOST_FILE_VERSION}.tar.gz...")
    file(ARCHIVE_EXTRACT INPUT "${OCTANE_EXTERNAL_DIR}/lib/boost_${OCTANE_BOOST_FILE_VERSION}.tar.gz" DESTINATION "${OCTANE_EXTERNAL_DIR}/lib")
endif()

file(LOCK "${OCTANE_EXTERNAL_DIR}/lib/boost.lock" RELEASE)

add_library(Boost::boost INTERFACE IMPORTED GLOBAL)
target_include_directories(Boost::boost SYSTEM INTERFACE ${boost_SOURCE_DIR})
target_compile_definitions(
        Boost::boost
        INTERFACE
        BOOST_ALL_NO_LIB=1
        BOOST_NO_EXCEPTIONS=1
)

file(GLOB Boost_lib_dirs RELATIVE ${boost_SOURCE_DIR}/libs ${boost_SOURCE_DIR}/libs/*)

foreach(lib ${Boost_lib_dirs})
    if (IS_DIRECTORY ${boost_SOURCE_DIR}/libs/${lib})
        file(GLOB Boost_lib_srcs RELATIVE ${boost_SOURCE_DIR}/libs/${lib} ${boost_SOURCE_DIR}/libs/${lib}/*)

        foreach(dir ${Boost_lib_srcs})
            if (IS_DIRECTORY ${boost_SOURCE_DIR}/libs/${lib}/${dir})
                if (${dir} STREQUAL "src")
                    add_boost_lib(TARGET ${lib})
                endif()
            endif()
        endforeach()
    endif()
endforeach()
