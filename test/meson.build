foreach testname: ['api', 'api-fill', 'stress']
    exe = executable(
        testname,
        'test-@0@.c'.format(testname),
        dependencies: mimalloc_dep,
    )
    test(testname, exe)
endforeach

executable('dynamic-override', 'main-override.c', dependencies: mimalloc_dep)
executable('dynamic-override-cxx', 'main-override.cpp', dependencies: mimalloc_dep)
if get_option('default_library') != 'shared'
    executable('static-override', 'main-override.c', dependencies: mimalloc_static_dep)
    executable('static-override-cxx', 'main-override.cpp', dependencies: mimalloc_static_dep)
endif

executable('test-wrong', 'test-wrong.c', dependencies: mimalloc_dep)
