cmake_minimum_required(VERSION 3.10)
#It seems Cmake does not set default bild type so we force it
if( NOT CMAKE_BUILD_TYPE )
    set( CMAKE_BUILD_TYPE Debug CACHE STRING "Debug" FORCE )
endif()

project(freeopcua)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

find_package(ament_cmake REQUIRED)

option(BUILD_CLIENT "Build Client" ON)
option(BUILD_SERVER "Build Server" ON)

option(SSL_SUPPORT_MBEDTLS "Support rsa-oaep password encryption using mbedtls library " OFF)

option(BUILD_PYTHON "Build Python bindings" OFF)
option(BUILD_TESTING "Build and run tests" OFF)
option(BUILD_SHARED_LIBS "Build shared libraries." OFF)

set(CMAKE_CXX_FLAGS_DEBUG " -Wall -Wrange-loop-construct -ggdb -O0 ${CMAKE_CXX_FLAGS_DEBUG}")

SET(STATIC_LIBRARY_CXX_FLAGS)
SET(EXECUTABLE_CXX_FLAGS)
SET(DYNAMIC_LIBRARY_CXX_FLAGS)
SET(D -D)
set(CMAKE_CXX_FLAGS " -std=c++11 -Wall -fPIC ${CMAKE_CXX_FLAGS} ")
SET (CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS_INIT} $ENV{LDFLAGS}")
SET(DYNAMIC_ADDON_PATH "${CMAKE_LIBRARY_OUTPUT_DIRECTORY}/libtest_dynamic_addon.so")
SET(TEST_CORE_CONFIG_PATH "${PROJECT_SOURCE_DIR}/tests/core/configs/")
SET(OS_SUFFIX _lin)

#FIXME: remove that variable and link directly when necessary!!!!
SET(ADDITIONAL_LINK_LIBRARIES pthread dl ${LIBXML2_LIBRARIES})

find_package(spdlog)
if(spdlog_FOUND)
    message(STATUS "Found spdlog")
    set(ADDITIONAL_PUBLIC_COMPILE_OPTIONS -DHAVE_SYSTEM_SPDLOG)
    set(ADDITIONAL_LINK_LIBRARIES ${ADDITIONAL_LINK_LIBRARIES} spdlog::spdlog)
endif()

find_package(Boost COMPONENTS system program_options filesystem thread REQUIRED)
include_directories(${Boost_INCLUDE_DIRS})
link_directories(${Boost_LIBRARY_DIRS})
message(STATUS "Boost INCLUDE DIR IS: " ${Boost_INCLUDE_DIRS})
message(STATUS "Boost LIBRARY DIR IS: " ${Boost_LIBRARY_DIRS})

message(STATUS "SSL support using libmbedtls: " ${SSL_SUPPORT_MBEDTLS})
if (SSL_SUPPORT_MBEDTLS)
    add_definitions(-DSSL_SUPPORT_MBEDTLS)
    IF (DEFINED SSL_SUPPORT_MBEDTLS_INCLUDE_DIR)
        message(STATUS "  mbedtls library INCLUDE_DIR: " ${SSL_SUPPORT_MBEDTLS_INCLUDE_DIR})
        include_directories( ${SSL_SUPPORT_MBEDTLS_INCLUDE_DIR} )
    ENDIF ()
    IF (DEFINED SSL_SUPPORT_MBEDTLS_LIB_DIR)
        message(STATUS "  mbedtls library LIB_DIR: " ${SSL_SUPPORT_MBEDTLS_LIB_DIR})
        link_directories( ${SSL_SUPPORT_MBEDTLS_LIB_DIR} )
    ENDIF ()
    SET(SSL_SUPPORT_LINK_LIBRARIES mbedcrypto mbedx509 mbedtls)
    message(STATUS "  mbedtls LIBRARIES: " ${SSL_SUPPORT_LINK_LIBRARIES})
endif (SSL_SUPPORT_MBEDTLS)


#include_directories(${PROJECT_SOURCE_DIR}/include)

############################################################################
# Protocol library
############################################################################

ADD_CUSTOM_COMMAND(
    OUTPUT ${PROJECT_SOURCE_DIR}/include/opc/ua/protocol/object_ids.h
    COMMAND python ${PROJECT_SOURCE_DIR}/schemas/codegen.py cxx object_ids > ${PROJECT_SOURCE_DIR}/include/opc/ua/protocol/object_ids.h
)


add_library(opcuaprotocol
    src/protocol/rawsize_auto.cpp
    src/protocol/serialize_auto.cpp
    src/protocol/deserialize_auto.cpp
    src/protocol/constructors_auto.cpp
    src/protocol/protocol.cpp
    src/protocol/binary_attribute.cpp
    src/protocol/binary_data_value.cpp
    src/protocol/binary_endpoints.cpp
    src/protocol/binary_messages.cpp
    src/protocol/binary_node_management.cpp
    src/protocol/binary_raw_size.cpp
    src/protocol/binary_serialization.h
    src/protocol/binary_session.cpp
    src/protocol/binary_stream.cpp
    src/protocol/binary_variant.cpp
    src/protocol/binary_view.cpp
    src/protocol/input_from_buffer.cpp
    src/protocol/monitored_items.cpp
    src/protocol/nodeid.cpp
    src/protocol/session.cpp
    src/protocol/status_codes.cpp
    src/protocol/string_utils.cpp
    src/protocol/string_utils_attributeid_tostring.cpp
    src/protocol/string_utils_objectid_tostring.cpp
    src/protocol/string_utils_statuscode_tostring.cpp
    src/protocol/subscriptions.cpp
)

target_compile_options(opcuaprotocol PUBLIC ${STATIC_LIBRARY_CXX_FLAGS})
target_compile_options(opcuaprotocol PUBLIC ${ADDITIONAL_PUBLIC_COMPILE_OPTIONS})
target_link_libraries(opcuaprotocol ${ADDITIONAL_LINK_LIBRARIES})
target_include_directories(opcuaprotocol PUBLIC $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include> $<INSTALL_INTERFACE:include/>)
target_link_libraries(opcuaprotocol ${Boost_LIBRARIES})


############################################################################
# core library
############################################################################

SET(opcuacore_SOURCES
    src/core/common/addons_core/addon_manager.cpp
    src/core/common/addons_core/config_file.cpp
    src/core/common/addons_core/dynamic_library${OS_SUFFIX}.cpp
    src/core/common/addons_core/dynamic_addon_factory.cpp
    src/core/common/addons_core/dynamic_library.h
    src/core/common/addons_core/errors_addon_manager.cpp
    src/core/common/common_errors.cpp
    src/core/common/exception.cpp
    src/core/common/thread.cpp
    src/core/common/uri_facade${OS_SUFFIX}${NO_REGEX_SUFFIX}.cpp
    src/core/common/value.cpp
    src/core/event.cpp
    src/core/model_impl.h
    src/core/model_node.cpp
    src/core/model_object.cpp
    src/core/model_object_type.cpp
    src/core/model_server.cpp
    src/core/model_variable.cpp
    src/core/node.cpp
    src/core/opcua_errors.cpp
    src/core/socket_channel.cpp
    src/core/subscription.cpp
    src/core/server_operations.cpp
)

add_library(opcuacore ${opcuacore_SOURCES})

target_compile_options(opcuacore PUBLIC ${STATIC_LIBRARY_CXX_FLAGS})
target_compile_options(opcuacore PUBLIC ${ADDITIONAL_PUBLIC_COMPILE_OPTIONS})
target_link_libraries(opcuacore ${ADDITIONAL_LINK_LIBRARIES} opcuaprotocol ${Boost_SYSTEM_LIBRARY} ${Boost_FILESYSTEM_LIBRARY})
target_include_directories(opcuacore PUBLIC $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include> $<INSTALL_INTERFACE:include/>)

############################################################################
# client library
############################################################################
add_library(opcuaclient
    src/client/binary_connection.cpp
    src/client/binary_client.cpp
    src/client/binary_client_addon.cpp
    src/client/client.cpp
)

target_compile_options(opcuaclient PUBLIC ${STATIC_LIBRARY_CXX_FLAGS})
target_compile_options(opcuaclient PUBLIC ${ADDITIONAL_PUBLIC_COMPILE_OPTIONS})
target_link_libraries(opcuaclient
    opcuacore
    ${ADDITIONAL_LINK_LIBRARIES}
    ${Boost_PROGRAMOPTIONS_LIBRARY}
    ${SSL_SUPPORT_LINK_LIBRARIES}
)

target_include_directories(opcuaclient PUBLIC $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include> $<INSTALL_INTERFACE:include/>)


############################################################################
# server library
############################################################################

add_library(opcuaserver
    src/server/address_space_addon.cpp
    src/server/address_space_internal.cpp
    src/server/asio_addon.cpp
    src/server/common_addons.cpp
    src/server/endpoints_parameters.cpp
    src/server/endpoints_registry.cpp
    src/server/endpoints_services_addon.cpp
    src/server/internal_subscription.cpp
    src/server/server.cpp
    src/server/opc_tcp_async.cpp
    src/server/opc_tcp_async_addon.cpp
    src/server/opc_tcp_async_parameters.cpp
    src/server/opc_tcp_processor.cpp
    src/server/tcp_server.cpp
    src/server/server_object.cpp
    src/server/server_object_addon.cpp
    src/server/services_registry_factory.cpp
    src/server/services_registry_impl.cpp
    src/server/standard_address_space_part3.cpp
    src/server/standard_address_space_part4.cpp
    src/server/standard_address_space_part5.cpp
    src/server/standard_address_space_part8.cpp
    src/server/standard_address_space_part9.cpp
    src/server/standard_address_space_part10.cpp
    src/server/standard_address_space_part11.cpp
    src/server/standard_address_space_part13.cpp
    src/server/standard_address_space.cpp
    src/server/standard_address_space_addon.cpp
    src/server/subscription_service_addon.cpp
    src/server/subscription_service_internal.cpp
    )

if (NOT CMAKE_VERSION VERSION_LESS 2.8.12)
    target_compile_options(opcuaserver PUBLIC ${STATIC_LIBRARY_CXX_FLAGS})
endif ()
target_compile_options(opcuaserver PUBLIC ${ADDITIONAL_PUBLIC_COMPILE_OPTIONS})
target_link_libraries(opcuaserver ${ADDITIONAL_LINK_LIBRARIES} opcuacore opcuaprotocol ${Boost_SYSTEM_LIBRARY})
target_include_directories(opcuaserver PUBLIC $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include> $<INSTALL_INTERFACE:include/>)


############################################################################
# ros install
############################################################################
ament_export_targets(${PROJECT_NAME} HAS_LIBRARY_TARGET)
ament_export_dependencies()

install(
        DIRECTORY include/
        DESTINATION include
)

install(TARGETS opcuaserver opcuaclient opcuacore opcuaprotocol EXPORT ${PROJECT_NAME} DESTINATION lib/${PROJECT_NAME})
#install(TARGETS opcuaclient EXPORT ${PROJECT_NAME} DESTINATION lib/${PROJECT_NAME})
#install(TARGETS opcuacore EXPORT ${PROJECT_NAME} DESTINATION lib/${PROJECT_NAME})
#install(TARGETS opcuaprotocol EXPORT ${PROJECT_NAME} DESTINATION lib/${PROJECT_NAME})

ament_package()
