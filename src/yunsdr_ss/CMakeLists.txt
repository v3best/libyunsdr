CMAKE_MINIMUM_REQUIRED(VERSION 2.8)
SET(LIBRARY_OUTPUT_PATH "${PROJECT_BINARY_DIR}/lib")
#AUX_SOURCE_DIRECTORY(${PROJECT_SOURCE_DIR}/src/yunsdr_ss/src YUNSDR_SS_src)
INCLUDE_DIRECTORIES("${PROJECT_SOURCE_DIR}/src/yunsdr_ss/include")

SET(YUNSDR_SS_src "")
list(APPEND YUNSDR_SS_src src/debug.c src/spinlock.c src/transport.c  src/yunsdr_api_ss.c)


########################################################################
# Find dependencies
########################################################################
if(ENABLE_PCIE)
	if(RIFFA_FOUND)
		add_definitions(-DENABLE_PCIE)
		list(APPEND YUNSDR_SS_src src/interface_pcie.c)
		include_directories(${RIFFA_INCLUDE_DIRS})
		link_directories(${RIFFA_LIBRARY_DIRS})
	endif(RIFFA_FOUND)
endif(ENABLE_PCIE)

if (ENABLE_SFP)
	add_definitions(-DENABLE_SFP)
	list(APPEND YUNSDR_SS_src src/interface_sfp.c)
endif(ENABLE_SFP)


ADD_LIBRARY(yunsdr_ss SHARED ${YUNSDR_SS_src})
ADD_LIBRARY(yunsdr_ss_static STATIC ${YUNSDR_SS_src})
SET_TARGET_PROPERTIES(yunsdr_ss_static PROPERTIES OUTPUT_NAME "yunsdr_ss")

if(RIFFA_FOUND)
	target_link_libraries(yunsdr_ss ${RIFFA_LIBRARIES})
	target_link_libraries(yunsdr_ss_static ${RIFFA_LIBRARIES})
endif(RIFFA_FOUND)

if (HAVE_AVX2)
	if(MSVC)
		set_target_properties(yunsdr_ss PROPERTIES COMPILE_FLAGS "/arch:AVX2 /MT")
		set_target_properties(yunsdr_ss_static PROPERTIES COMPILE_FLAGS "/arch:AVX2 /MT")
		add_definitions(-DLV_HAVE_AVX2 -DLV_HAVE_AVX -DLV_HAVE_SSE)
	else(MSVC)
		set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -mfpmath=sse -mavx2 -DLV_HAVE_AVX2 -DLV_HAVE_AVX -DLV_HAVE_SSE")
	endif(MSVC)
else (HAVE_AVX2)
	if(HAVE_AVX)
		if(MSVC)
			add_definitions(-DLV_HAVE_AVX -DLV_HAVE_SSE)
			set_target_properties(yunsdr_ss PROPERTIES COMPILE_FLAGS "/arch:AVX /MT")
			set_target_properties(yunsdr_ss_static PROPERTIES COMPILE_FLAGS "/arch:AVX /MT")
		else(MSVC)
			set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -mfpmath=sse -mavx -DLV_HAVE_AVX -DLV_HAVE_SSE")
		endif(MSVC)
	elseif(HAVE_SSE)
		if(MSVC)
			add_definitions(-DLV_HAVE_SSE)
			set_target_properties(yunsdr_ss PROPERTIES COMPILE_FLAGS "/arch:SSE2 /MT")
			set_target_properties(yunsdr_ss_static PROPERTIES COMPILE_FLAGS "/arch:SSE2 /MT")
		else(MSVC)
			set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -mfpmath=sse -msse4.1 -DLV_HAVE_SSE")
		endif(MSVC)
	endif(HAVE_AVX)
endif (HAVE_AVX2)

if(MSVC)
	add_definitions(-D_WINSOCK_DEPRECATED_NO_WARNINGS)
	add_definitions(-D_CRT_SECURE_NO_WARNINGS)
endif(MSVC)

if(MINGW)
    ADD_CUSTOM_COMMAND(TARGET yunsdr_ss
        POST_BUILD
        COMMAND cd ${LIBRARY_OUTPUT_PATH} & gendef libyunsdr_ss.dll
        COMMENT "Generate def file for libyunsdr_ss.dll"
    )
    INSTALL(FILES ${LIBRARY_OUTPUT_PATH}/libyunsdr_ss.def
		DESTINATION ${CMAKE_INSTALL_PREFIX}/lib)
endif(MINGW)

if(MSVC OR MINGW)
	target_link_libraries(yunsdr_ss ws2_32)
	target_link_libraries(yunsdr_ss_static ws2_32)
	target_link_libraries(yunsdr_ss ${RIFFA_LIBRARIES})
	target_link_libraries(yunsdr_ss_static ${RIFFA_LIBRARIES})

	set(CMAKE_STATIC_LIBRARY_PREFIX "lib")
	set(CMAKE_SHARED_LIBRARY_PREFIX "lib")

	INSTALL(TARGETS yunsdr_ss yunsdr_ss_static
		RUNTIME DESTINATION ${CMAKE_INSTALL_PREFIX}/lib
		LIBRARY DESTINATION ${CMAKE_INSTALL_PREFIX}/lib
		ARCHIVE DESTINATION ${CMAKE_INSTALL_PREFIX}/lib)

	INSTALL(FILES ${PROJECT_SOURCE_DIR}/src/yunsdr_ss/include/yunsdr_api_ss.h 
		DESTINATION ${CMAKE_INSTALL_PREFIX}/include)
else(MSVC OR MINGW)
	target_link_libraries(yunsdr_ss pthread m)
	target_link_libraries(yunsdr_ss_static pthread m)
	INSTALL(TARGETS yunsdr_ss yunsdr_ss_static
		LIBRARY DESTINATION /usr/local/lib
		ARCHIVE DESTINATION /usr/local/lib)

	INSTALL(FILES ${PROJECT_SOURCE_DIR}/src/yunsdr_ss/include/yunsdr_api_ss.h 
		DESTINATION /usr/local/include)
endif(MSVC OR MINGW)


ADD_SUBDIRECTORY(tests)
