# Set the example directory path
SET(VQ_EXAMPLE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/)
SET(VQ_BINARY_DIR ${CMAKE_CURRENT_BINARY_DIR}/../src/)
SET(QUAKELIB_BINARY_DIR ${CMAKE_CURRENT_BINARY_DIR}/../quakelib/)

# Copy the fault traces to the build directory for use in the tests
FILE(COPY ${VQ_EXAMPLE_DIR}/fault_traces DESTINATION ${CMAKE_CURRENT_BINARY_DIR}/../)
FILE(COPY ${VQ_EXAMPLE_DIR}/ca_model DESTINATION ${CMAKE_CURRENT_BINARY_DIR}/../)

SET(TAPER_METHODS none taper taper_renorm)
SET(METHOD_MEANSLIP 100.4 0 0)
SET(METHOD_INTEREVENT 49.2 0 0)

SET(MAX_TIME 180)

INCLUDE(FindPythonModule)
FIND_PYTHON_MODULE(h5py)

FIND_PACKAGE(SWIG)

# If MPI is available, set up MPI based tests as well
IF(DEFINED MPI_C_FOUND AND MPI_CXX_FOUND)
    SET(NUM_PROCS 1 2 4)
ELSE(DEFINED MPI_C_FOUND AND MPI_CXX_FOUND)
    SET(NUM_PROCS 1)
ENDIF(DEFINED MPI_C_FOUND AND MPI_CXX_FOUND)

SET(SETUP_PARAMS_SCRIPT ${VQ_EXAMPLE_DIR}/setup_params.sh)
SET(CHECK_SCRIPT ${VQ_EXAMPLE_DIR}/check_results.py)
SET(GREENS_SCRIPT ${VQ_EXAMPLE_DIR}/sum_greens.py)
SET(ENV{PATH} $ENV{PATH}:${CMAKE_CURRENT_BINARY_DIR}/../src/)

LIST(LENGTH TAPER_METHODS N)
MATH(EXPR NUM_TAPER "${N}-1")
 
# --------------  Single Fault Section Tests ----------------------------
FOREACH(NPROC ${NUM_PROCS})
    # Don't use 12k meshing (i.e. single element) if more than 1 proc (otherwise VQ will fail)
    IF (NPROC EQUAL 1)
        SET(RESOLUTIONS     12000 6000 4000 3000 2000)
    ELSE (NPROC EQUAL 1)
        SET(RESOLUTIONS     6000 4000 3000 2000)
    ENDIF (NPROC EQUAL 1)

    FOREACH(TAPER_IND RANGE ${NUM_TAPER})
        LIST(GET TAPER_METHODS ${TAPER_IND} TAPER)
        #LIST(GET METHOD_MEANSLIP ${TAPER_IND} MEANSLIP)
        #LIST(GET METHOD_INTEREVENT ${TAPER_IND} INTEREVENT)
        FOREACH(RES ${RESOLUTIONS})
            SET(TEST_DIR ${CMAKE_CURRENT_BINARY_DIR}/PROCS${NPROC}/${TAPER}/)
            FILE(MAKE_DIRECTORY ${TEST_DIR})
            SET(EVENT_FILE     ${TEST_DIR}events_${RES}.txt)
            SET(SWEEP_FILE     ${TEST_DIR}sweeps_${RES}.txt)
            

            # Test that the mesher works and generate a parameter file
            ADD_TEST(
                NAME mesh_P${NPROC}_${TAPER}_${RES} WORKING_DIRECTORY ${TEST_DIR}
                COMMAND mesher
                --import_file=../../../fault_traces/single_fault_trace.txt
                --import_file_type=trace --import_trace_element_size=${RES}
                --taper_fault_method=${TAPER}
                --export_file=single_fault_${RES}.txt
                --export_file_type=text
                --export_file=single_fault_${RES}.kml
                --export_file_type=kml
                --print_statistics=statistics_${RES}.txt
                )
            ADD_TEST(NAME param_P${NPROC}_${TAPER}_${RES} WORKING_DIRECTORY ${TEST_DIR}
                COMMAND ${SETUP_PARAMS_SCRIPT} ${RES} 1.0 single_fault ${VQ_EXAMPLE_DIR}/example_params.prm params_${RES}.prm)
            SET_TESTS_PROPERTIES (param_P${NPROC}_${TAPER}_${RES} PROPERTIES DEPENDS mesh_P${NPROC}_${TAPER}_${RES} TIMEOUT ${MAX_TIME})

            # Test that the simulation runs with the resulting mesh
            IF (NPROC EQUAL 1)
                ADD_TEST(NAME run_P${NPROC}_${TAPER}_${RES} WORKING_DIRECTORY ${TEST_DIR}
                    COMMAND ${VQ_BINARY_DIR}/vq params_${RES}.prm)
            ELSE (NPROC EQUAL 1)
                ADD_TEST(NAME run_P${NPROC}_${TAPER}_${RES} WORKING_DIRECTORY ${TEST_DIR}
                    COMMAND ${MPIEXEC} ${MPIEXEC_NUMPROC_FLAG} ${NPROC} ${VQ_BINARY_DIR}/vq params_${RES}.prm)
            ENDIF (NPROC EQUAL 1)
            SET_TESTS_PROPERTIES (run_P${NPROC}_${TAPER}_${RES} PROPERTIES DEPENDS param_P${NPROC}_${TAPER}_${RES} TIMEOUT ${MAX_TIME})

            # Check that the results are internally self consistent
            IF(DEFINED SWIG_FOUND)
                ADD_TEST(NAME test_consistent_P${NPROC}_${TAPER}_${RES} WORKING_DIRECTORY ${TEST_DIR}
                    COMMAND ${PYTHON_EXECUTABLE} ${CHECK_SCRIPT} --event_file ${EVENT_FILE} --sweep_file ${SWEEP_FILE} --check_consistent)
                SET_TESTS_PROPERTIES (test_consistent_P${NPROC}_${TAPER}_${RES} PROPERTIES
                    DEPENDS run_P${NPROC}_${TAPER}_${RES}
                    ENVIRONMENT "PYTHONPATH=${QUAKELIB_BINARY_DIR}/python/"
                    TIMEOUT ${MAX_TIME})
            ENDIF(DEFINED SWIG_FOUND)

            # Check that the mean slip is near our expectations
            # Schultz: For now removing until we come up with better tests
            #IF (NOT ${MEANSLIP} EQUAL 0)
            #    ADD_TEST(NAME test_slip_P${NPROC}_${TAPER}_${RES} WORKING_DIRECTORY ${TEST_DIR}
            #        COMMAND ${PYTHON_EXECUTABLE} ${CHECK_SCRIPT} --event_file ${EVENT_FILE} --sweep_file ${SWEEP_FILE} --mean_slip ${MEANSLIP})
            #    SET_TESTS_PROPERTIES (test_slip_P${NPROC}_${TAPER}_${RES} PROPERTIES
            #        DEPENDS run_P${NPROC}_${TAPER}_${RES}
            #        ENVIRONMENT "PYTHONPATH=${QUAKELIB_BINARY_DIR}/python/"
            #        TIMEOUT ${MAX_TIME})
            #ENDIF (NOT ${MEANSLIP} EQUAL 0)

            # Check that the mean interevent time is near our expectations
            # Schultz: For now removing until we come up with better tests
            #IF (NOT ${INTEREVENT} EQUAL 0)
            #    ADD_TEST(NAME test_interevent_P${NPROC}_${TAPER}_${RES} WORKING_DIRECTORY ${TEST_DIR}
            #        COMMAND ${PYTHON_EXECUTABLE} ${CHECK_SCRIPT} --event_file ${EVENT_FILE} --sweep_file ${SWEEP_FILE} --mean_interevent ${INTEREVENT})
            #    SET_TESTS_PROPERTIES (test_interevent_P${NPROC}_${TAPER}_${RES} PROPERTIES
            #        DEPENDS run_P${NPROC}_${TAPER}_${RES}
            #        ENVIRONMENT "PYTHONPATH=${QUAKELIB_BINARY_DIR}/python/"
            #        TIMEOUT ${MAX_TIME})
            #ENDIF(NOT ${INTEREVENT} EQUAL 0) 
        ENDFOREACH(RES ${RESOLUTIONS})
    ENDFOREACH(TAPER_IND RANGE ${NUM_TAPER})
ENDFOREACH(NPROC ${NUM_PROCS})




# --------------  Two Separate Faults  -------------------

SET(RESOLUTIONS_TWO 6000 3000)
SET(METHOD_INTEREVENT_TWO 50 0 0)

FOREACH(TAPER_IND RANGE ${NUM_TAPER})
    LIST(GET TAPER_METHODS ${TAPER_IND} TAPER)
    LIST(GET METHOD_MEANSLIP ${TAPER_IND} MEANSLIP)
    LIST(GET METHOD_INTEREVENT_TWO ${TAPER_IND} INTEREVENT)
    FOREACH(RES ${RESOLUTIONS_TWO})
        SET(TEST_DIR ${CMAKE_CURRENT_BINARY_DIR}/two_fault/${TAPER}/)
        FILE(MAKE_DIRECTORY ${TEST_DIR})
        SET(EVENT_FILE_TWO ${TEST_DIR}events_${RES}.txt)
        SET(SWEEP_FILE_TWO ${TEST_DIR}sweeps_${RES}.txt)


        # Test that the mesher works and generate a parameter file
        ADD_TEST(
            NAME mesh_two_${TAPER}_${RES} WORKING_DIRECTORY ${TEST_DIR}
            COMMAND mesher
            --import_file=../../../fault_traces/two_fault_1.txt
            --import_file_type=trace --import_trace_element_size=${RES}
            --taper_fault_method=${TAPER}
            --import_file=../../../fault_traces/two_fault_2.txt
            --import_file_type=trace --import_trace_element_size=${RES}
            --taper_fault_method=${TAPER}
            --export_file=two_fault_${RES}.txt
            --export_file_type=text
            --export_file=two_fault_${RES}.kml
            --export_file_type=kml
            --print_statistics=statistics_${RES}.txt
            )
 
        ADD_TEST(NAME param_two_${TAPER}_${RES} WORKING_DIRECTORY ${TEST_DIR}
            COMMAND ${SETUP_PARAMS_SCRIPT} ${RES} 0.7 two_fault ${VQ_EXAMPLE_DIR}/example_params.prm params_${RES}.prm)
        SET_TESTS_PROPERTIES (param_two_${TAPER}_${RES} PROPERTIES DEPENDS mesh_two_${TAPER}_${RES} TIMEOUT ${MAX_TIME})

        # Test that the simulation runs with the resulting mesh
        ADD_TEST(NAME run_two_${TAPER}_${RES} WORKING_DIRECTORY ${TEST_DIR}
            COMMAND vq params_${RES}.prm)
        SET_TESTS_PROPERTIES (run_two_${TAPER}_${RES} PROPERTIES DEPENDS param_two_${TAPER}_${RES} TIMEOUT ${MAX_TIME})

	IF(DEFINED SWIG_FOUND)
            # Check that the results are internally self consistent
            ADD_TEST(NAME test_two_consistent_${TAPER}_${RES} WORKING_DIRECTORY ${TEST_DIR}
                COMMAND ${PYTHON_EXECUTABLE} ${CHECK_SCRIPT} --event_file ${EVENT_FILE_TWO} --sweep_file ${SWEEP_FILE_TWO} --check_consistent)
            SET_TESTS_PROPERTIES (test_two_consistent_${TAPER}_${RES} PROPERTIES
                DEPENDS run_two_${TAPER}_${RES}
                ENVIRONMENT "PYTHONPATH=${QUAKELIB_BINARY_DIR}/python/"
                TIMEOUT ${MAX_TIME})
        ENDIF(DEFINED SWIG_FOUND)

        # Check that the mean slip is near our expectations
        #    These two faults have same slip rate as single fault, so using
        #       same mean slip values
        #IF (NOT ${MEANSLIP} EQUAL 0)
        #    ADD_TEST(NAME test_two_slip_${TAPER}_${RES} WORKING_DIRECTORY ${TEST_DIR}
        #        COMMAND ${PYTHON_EXECUTABLE} ${CHECK_SCRIPT} --event_file ${EVENT_FILE_TWO} --sweep_file ${SWEEP_FILE_TWO} --mean_slip ${MEANSLIP})
        #    SET_TESTS_PROPERTIES (test_two_slip_${TAPER}_${RES} PROPERTIES
        #        DEPENDS run_two_${TAPER}_${RES}
        #        ENVIRONMENT "PYTHONPATH=${QUAKELIB_BINARY_DIR}/python/"
        #        TIMEOUT ${MAX_TIME})
        #ENDIF (NOT ${MEANSLIP} EQUAL 0)

        # Check that the mean interevent time is near our expectations
        # TODO: These tests fail, Kasey needs to figure out why
        #IF (NOT ${INTEREVENT} EQUAL 0)
        #    ADD_TEST(NAME test_two_interevent_${TAPER}_${RES} WORKING_DIRECTORY ${TEST_DIR}
        #        COMMAND ${PYTHON_EXECUTABLE} ${CHECK_SCRIPT} --event_file ${EVENT_FILE_TWO} --sweep_file ${SWEEP_FILE_TWO} --mean_interevent ${INTEREVENT})
        #    SET_TESTS_PROPERTIES (test_two_interevent_${TAPER}_${RES} PROPERTIES
        #       DEPENDS run_two_${TAPER}_${RES}
        #       ENVIRONMENT "PYTHONPATH=${QUAKELIB_BINARY_DIR}/python/"
        #       TIMEOUT ${MAX_TIME})
        #ENDIF(NOT ${INTEREVENT} EQUAL 0)             
    ENDFOREACH(RES ${RESOLUTIONS_TWO})
ENDFOREACH(TAPER_IND RANGE ${NUM_TAPER})


# Confirm HDF5 Greens file output and input works correctly on one or more processors
IF (HDF5_FOUND)
    FOREACH(NPROC ${NUM_PROCS})
        SET(TEST_DIR ${CMAKE_CURRENT_BINARY_DIR}/GREENS_P${NPROC}/)
        SET(RES 3000)
        FILE(MAKE_DIRECTORY ${TEST_DIR})
        SET(TEST_SUFFIX P${NPROC}_green_${RES})
        SET(EVENT_FILE ${TEST_DIR}events_${RES}.txt)
        SET(SWEEP_FILE ${TEST_DIR}sweeps_${RES}.txt)

        ADD_TEST(
            NAME mesh_${TEST_SUFFIX} WORKING_DIRECTORY ${TEST_DIR}
            COMMAND mesher
            --import_file=../../fault_traces/single_fault_trace.txt
            --import_file_type=trace --import_trace_element_size=${RES}
            --taper_fault_method=none
            --export_file=single_fault_${RES}.txt
            --export_file_type=text
            )
        ADD_TEST(NAME param_gen_${TEST_SUFFIX} WORKING_DIRECTORY ${TEST_DIR}
            COMMAND ${SETUP_PARAMS_SCRIPT} ${RES} 0.5 single_fault ${VQ_EXAMPLE_DIR}/greens_generate.prm params_gen_${RES}.prm)
        SET_TESTS_PROPERTIES (param_gen_${TEST_SUFFIX} PROPERTIES DEPENDS mesh_${TEST_SUFFIX} TIMEOUT ${MAX_TIME})

        ADD_TEST(NAME param_run_${TEST_SUFFIX} WORKING_DIRECTORY ${TEST_DIR}
            COMMAND ${SETUP_PARAMS_SCRIPT} ${RES} 0.5 single_fault ${VQ_EXAMPLE_DIR}/greens_input.prm params_run_${RES}.prm)
        SET_TESTS_PROPERTIES (param_run_${TEST_SUFFIX} PROPERTIES DEPENDS param_gen_${TEST_SUFFIX} TIMEOUT ${MAX_TIME})

        ADD_TEST(NAME run_gen_${TEST_SUFFIX} WORKING_DIRECTORY ${TEST_DIR}
            COMMAND ${MPIEXEC} ${MPIEXEC_NUMPROC_FLAG} ${NPROC} ${VQ_BINARY_DIR}/vq params_gen_${RES}.prm)
        SET_TESTS_PROPERTIES (run_gen_${TEST_SUFFIX} PROPERTIES DEPENDS param_run_${TEST_SUFFIX} TIMEOUT ${MAX_TIME})

        # If h5py is available, confirm the values in the Greens file sum to the correct value
        IF(PY_H5PY)
            ADD_TEST(NAME check_sum_${TEST_SUFFIX} WORKING_DIRECTORY ${TEST_DIR}
                COMMAND ${PYTHON_EXECUTABLE} ${GREENS_SCRIPT} greens_3000.h5 6.9056016275796917e-08 -91753588.690448046)
            SET_TESTS_PROPERTIES (check_sum_${TEST_SUFFIX} PROPERTIES DEPENDS run_gen_${TEST_SUFFIX} TIMEOUT ${MAX_TIME})
        ENDIF(PY_H5PY)

        # Test that the simulation runs with the resulting mesh
        ADD_TEST(NAME run_full_${TEST_SUFFIX} WORKING_DIRECTORY ${TEST_DIR}
            COMMAND ${MPIEXEC} ${MPIEXEC_NUMPROC_FLAG} ${NPROC} ${VQ_BINARY_DIR}/vq params_run_${RES}.prm)
        SET_TESTS_PROPERTIES (run_full_${TEST_SUFFIX} PROPERTIES DEPENDS run_gen_${TEST_SUFFIX} TIMEOUT ${MAX_TIME})

	IF(DEFINED SWIG_FOUND)
            # Check that the results are internally self consistent
            ADD_TEST(NAME check_consistent_${TEST_SUFFIX} WORKING_DIRECTORY ${TEST_DIR}
                COMMAND ${PYTHON_EXECUTABLE} ${CHECK_SCRIPT} --event_file ${EVENT_FILE} --sweep_file ${SWEEP_FILE} --check_consistent)
            SET_TESTS_PROPERTIES (check_consistent_${TEST_SUFFIX} PROPERTIES
                DEPENDS run_full_${TEST_SUFFIX}
                ENVIRONMENT "PYTHONPATH=${QUAKELIB_BINARY_DIR}/python/"
                TIMEOUT ${MAX_TIME})
        ENDIF(DEFINED SWIG_FOUND)

        # Check that the mean slip is near our expectations
        #    These two faults have same slip rate as single fault, so using
        #       same mean slip values
	IF(DEFINED SWIG_FOUND)
            IF (NOT ${MEANSLIP} EQUAL 0)
                ADD_TEST(NAME check_slip_${TEST_SUFFIX} WORKING_DIRECTORY ${TEST_DIR}
                    COMMAND ${PYTHON_EXECUTABLE} ${CHECK_SCRIPT} --event_file ${EVENT_FILE} --sweep_file ${SWEEP_FILE} --mean_slip ${MEANSLIP})
                SET_TESTS_PROPERTIES (test_two_slip_${TAPER}_${RES} PROPERTIES
                    DEPENDS run_full_${TEST_SUFFIX}
                    ENVIRONMENT "PYTHONPATH=${QUAKELIB_BINARY_DIR}/python/"
                    TIMEOUT ${MAX_TIME})
            ENDIF (NOT ${MEANSLIP} EQUAL 0)
        ENDIF(DEFINED SWIG_FOUND)
    ENDFOREACH(NPROC ${NUM_PROCS})
ENDIF (HDF5_FOUND)


