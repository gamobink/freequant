cmake_minimum_required(VERSION 2.8)
project(freequant)

#set(BUILD_SHARED_LIBS true)
set(CMAKE_BUILD_TYPE Debug)
set(CMAKE_VERBOSE_MAKEFILE true)


list(APPEND CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR}/cmake)

#include(UseQt5)

if(WIN32)
    set(CMAKE_INSTALL_PREFIX $ENV{HOMEPATH})
else()
    set(CMAKE_INSTALL_PREFIX $ENV{HOME})
endif()

if(CMAKE_COMPILER_IS_GNUCXX)
    execute_process(COMMAND ${CMAKE_CXX_COMPILER} --version OUTPUT_VARIABLE CMAKE_CXX_COMPILER_VERSION OUTPUT_STRIP_TRAILING_WHITESPACE)
    string(REGEX MATCH "[0-9]+\\.[0-9]+\\.[0-9]+" CMAKE_CXX_COMPILER_VERSION "${CMAKE_CXX_COMPILER_VERSION}")
    if(${CMAKE_CXX_COMPILER_VERSION} VERSION_GREATER "4.6")
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11")
    else()
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++0x")
    endif()
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall -pthread")
endif(CMAKE_COMPILER_IS_GNUCXX)

if(MSVC AND MSVC_VERSION LESS 1700)
#    message(FATAL_ERROR "You should use VC 11 or above!")
endif()

if(CMAKE_CXX_COMPILER_ID STREQUAL "Clang")
    execute_process(COMMAND ${CMAKE_CXX_COMPILER} --version OUTPUT_VARIABLE CMAKE_CXX_COMPILER_VERSION OUTPUT_STRIP_TRAILING_WHITESPACE)
    string(REGEX MATCH "[0-9]+\\.[0-9]+" CMAKE_CXX_COMPILER_VERSION "${CMAKE_CXX_COMPILER_VERSION}")
    if(${CMAKE_CXX_COMPILER_VERSION} VERSION_GREATER "3.0")
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11")
    else()
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++0x")
    endif()
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall -pthread")
endif()

# Find external dependencies
if(WIN32)
    set(Boost_USE_STATIC_LIBS on)
endif(WIN32)
find_package(Boost REQUIRED COMPONENTS date_time chrono system filesystem thread unit_test_framework)
find_package(QuickFIX REQUIRED)
find_package(Doxygen)
if(DOXYGEN_FOUND)
#    add_subdirectory(doc)
endif()

if(WIN32 OR UNIX)
    option(BUILD_CTP "Build CTP." ON)
elseif(APPLE)
    option(BUILD_CTP "Build CTP." OFF)
endif()

option(FQ_ENABLE_SHARED OFF)

option(BUILD_FIX_PROVIDER "Build FIX Provider." ON)
option(BUILD_CTP_PROVIDER "Build CTP Provider." ON)
option(BUILD_CTPSSE_PROVIDER "Build CTPSSE Provider." ON)
option(BUILD_TWS_PROVIDER "Build TWS Provider." ON)

#option(BUILD_BINDINGS "Build bindings." ON)

# Find in-the-source dependencies
set(TWS_INCLUDE_DIRS ${PROJECT_SOURCE_DIR}/vendors/tws)

if(BUILD_CTP_PROVIDER)
    if(WIN32)
        set(CTP_INCLUDE_DIRS ${PROJECT_SOURCE_DIR}/vendors/ctp/api/trade/win/public)
        set(CTP_LIBRARY_DIRS ${PROJECT_SOURCE_DIR}/vendors/ctp/api/trade/win/lib ${PROJECT_SOURCE_DIR}/vendors/ctp/api/risk/lib)
    elseif(UNIX)
        set(CTP_INCLUDE_DIRS ${PROJECT_SOURCE_DIR}/vendors/ctp/api/trade/linux64/public)
        set(CTP_LIBRARY_DIRS ${PROJECT_SOURCE_DIR}/vendors/ctp/api/trade/linux64/lib)
    endif()

    if(WIN32)
        find_library(CTP_MDUSER_LIBRARY NAMES thostmduserapi PATHS ${CTP_LIBRARY_DIRS})
        find_library(CTP_TRADE_LIBRARY NAMES thosttraderapi PATHS ${CTP_LIBRARY_DIRS})
        find_library(CTP_RISK_LIBRARY NAMES riskuserapi PATHS ${CTP_LIBRARY_DIRS})
        if(CTP_MDUSER_LIBRARY AND CTP_TRADE_LIBRARY AND CTP_RISK_LIBRARY)
            set(CTP_LIBRARIES ${CTP_MDUSER_LIBRARY} ${CTP_TRADE_LIBRARY} ${CTP_RISK_LIBRARY})
            set(CTP_SHAREDLIBFILES ${PROJECT_SOURCE_DIR}/vendors/ctp/api/risk/lib/riskuserapi.dll
                    ${PROJECT_SOURCE_DIR}/vendors/ctp/api/trade/win/lib/thostmduserapi.dll
                    ${PROJECT_SOURCE_DIR}/vendors/ctp/api/trade/win/lib/thosttraderapi.dll
            )
        endif()
        set(WS2_32 ws2_32)
    elseif(UNIX)
        find_library(CTP_MDUSER_LIBRARY NAMES thostmduserapi PATHS ${CTP_LIBRARY_DIRS})
        find_library(CTP_TRADE_LIBRARY NAMES thosttraderapi PATHS ${CTP_LIBRARY_DIRS})
        if(CTP_MDUSER_LIBRARY AND CTP_TRADE_LIBRARY)
            set(CTP_LIBRARIES ${CTP_MDUSER_LIBRARY} ${CTP_TRADE_LIBRARY})
            set(CTP_SHAREDLIBFILES ${PROJECT_SOURCE_DIR}/vendors/ctp/api/trade/linux64/lib/libthostmduserapi.so
                ${PROJECT_SOURCE_DIR}/vendors/ctp/api/trade/linux64/lib/libthosttraderapi.so
            )
        endif()
    endif()
endif(BUILD_CTP_PROVIDER)

include_directories(${Boost_INCLUDE_DIRS} ${CURL_INCLUDE_DIRS} ${QUICKFIX_INCLUDE_DIRS} ${TWS_INCLUDE_DIRS} ${CTP_INCLUDE_DIRS})
include_directories(${PROJECT_SOURCE_DIR}/vendors/otl)
include_directories(${PROJECT_SOURCE_DIR}/src)
link_directories(${Boost_LIBRARY_DIRS})

add_subdirectory(src)
add_subdirectory(examples)
add_subdirectory(server)
#add_subdirectory(tools)

#add_subdirectory(swig)

include_directories(${PROJECT_BINARY_DIR})

option(BUILD_TESTS "Build tests." TRUE)
if (BUILD_TESTS)
    enable_testing()
#    add_subdirectory(tests)
endif()
if(WIN32)
    configure_file(${PROJECT_SOURCE_DIR}/vendors/ctp/api/trade/win/lib/thostmduserapi.dll ${PROJECT_BINARY_DIR}/bin/thostmduserapi.dll COPYONLY)
    configure_file(${PROJECT_SOURCE_DIR}/vendors/ctp/api/trade/win/lib/thosttraderapi.dll ${PROJECT_BINARY_DIR}/bin/thosttraderapi.dll COPYONLY)
    configure_file(${PROJECT_SOURCE_DIR}/vendors/ctp/api/risk/lib/riskuserapi.dll ${PROJECT_BINARY_DIR}/bin/riskuserapi.dll COPYONLY)
endif(WIN32)
