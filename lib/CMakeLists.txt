add_subdirectory(cpp)

option(WITH_THREADING "Include client library threading support?" ON)
if (${WITH_THREADING} STREQUAL ON)
	add_definitions("-DWITH_THREADING")
	if (WIN32)
		set (PTHREAD_LIBRARIES C:\\pthreads\\Pre-built.2\\lib\\x86\\pthreadVC2.lib)
		set (PTHREAD_INCLUDE_DIR C:\\pthreads\\Pre-built.2\\include)
	else (WIN32)
		find_library(LIBPTHREAD pthread)
		if (LIBPTHREAD)
			set (PTHREAD_LIBRARIES pthread)
		else (LIBPTHREAD)
			set (PTHREAD_LIBRARIES "")
		endif()
		set (PTHREAD_INCLUDE_DIR "")
	endif (WIN32)
else (${WITH_THREADING} STREQUAL ON)
	set (PTHREAD_LIBRARIES "")
	set (PTHREAD_INCLUDE_DIR "")
endif (${WITH_THREADING} STREQUAL ON)

include_directories(${mosquitto_SOURCE_DIR} ${mosquitto_SOURCE_DIR}/lib
			${STDBOOL_H_PATH} ${STDINT_H_PATH}
			${OPENSSL_INCLUDE_DIR} ${PTHREAD_INCLUDE_DIR})
link_directories(${mosquitto_SOURCE_DIR}/lib)

add_library(libmosquitto SHARED
	logging_mosq.c logging_mosq.h
	memory_mosq.c memory_mosq.h
	messages_mosq.c messages_mosq.h
	mosquitto.c mosquitto.h
	mosquitto_internal.h
	mqtt3_protocol.h
	net_mosq.c net_mosq.h
	read_handle.c read_handle.h
	read_handle_client.c
	read_handle_shared.c
	send_client_mosq.c
	send_mosq.c send_mosq.h
	socks_mosq.c
	srv_mosq.c
	thread_mosq.c
	time_mosq.c
	tls_mosq.c
	util_mosq.c util_mosq.h
	will_mosq.c will_mosq.h)

if(ANDROID)
    set (LIBRARIES ${PTHREAD_LIBRARIES})
else(ANDROID)
    set (LIBRARIES ${OPENSSL_LIBRARIES} ${PTHREAD_LIBRARIES})
endif(ANDROID)

if (UNIX AND NOT APPLE AND NOT ANDROID)
	find_library(LIBRT rt)
	if (LIBRT)
		set (LIBRARIES ${LIBRARIES} rt)
	endif (LIBRT)
endif (UNIX AND NOT APPLE AND NOT ANDROID)

if (WIN32)
	set (LIBRARIES ${LIBRARIES} ws2_32)
endif (WIN32)

if (${WITH_SRV} STREQUAL ON)
	# Simple detect c-ares
	find_path(ARES_HEADER ares.h)
	if (ARES_HEADER)
		add_definitions("-DWITH_SRV")
		set (LIBRARIES ${LIBRARIES} cares)
	else (ARES_HEADER)
		message(WARNING "c-ares library not found.")
	endif (ARES_HEADER)
endif (${WITH_SRV} STREQUAL ON)

if(ANDROID)
    add_library(libcrypto STATIC IMPORTED)
    set_target_properties(libcrypto PROPERTIES IMPORTED_LOCATION ${OPENSSL_LIBRARIES}/libcrypto.a)
    add_library(libssl STATIC IMPORTED)
    set_target_properties(libssl PROPERTIES IMPORTED_LOCATION ${OPENSSL_LIBRARIES}/libssl.a)
    target_link_libraries(libmosquitto ${LIBRARIES} libssl libcrypto)
else (ANDROID)
    target_link_libraries(libmosquitto ${LIBRARIES})
endif(ANDROID)

set_target_properties(libmosquitto PROPERTIES
	OUTPUT_NAME mosquitto
	VERSION ${VERSION}
	SOVERSION 1
)

install(TARGETS libmosquitto RUNTIME DESTINATION "${BINDIR}" LIBRARY DESTINATION "${LIBDIR}")

if(ANDROID)
	file(COPY mosquitto.h DESTINATION ${INCLUDEDIR})
else(ANDROID)
	install(FILES mosquitto.h DESTINATION "${INCLUDEDIR}")
endif(ANDROID)

if (UNIX)
	install(CODE "EXEC_PROGRAM(/sbin/ldconfig)")
endif (UNIX)
