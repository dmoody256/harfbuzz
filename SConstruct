import os
import sys
import glob
import re
import time
import datetime
import atexit
import platform
import fileinput
import subprocess

import SCons.Action
import SCons.Script.Main

from BuildUtils.SconsUtils import SetBuildJobs, SetupBuildEnv, ProgressCounter
from BuildUtils.ColorPrinter import ColorPrinter
from BuildUtils.FindPackages import FindFreetype, FindGraphite2, FindGlib, FindIcu
from BuildUtils.ConfigureChecks import *


def CreateNewEnv():

    p = ColorPrinter()
    SetupOptions()

    harfbuzz_options = [
        'option_have_freetype',
        'option_have_graphite2',
        'option_have_glib',
        'option_have_icu',
        'option_have_gobject',
        'option_have_introspection',
        'option_build_subset',
        'option_build_tests',
        'option_build_shared',
        'option_build_ucdn',
        'option_build_utils',
    ]

    if sys.platform == 'win32':
        harfbuzz_options += [
            'option_have_uniscribe',
            'option_have_directwrite',
        ]
           
    if sys.platform == 'darwin':
        harfbuzz_options += [
            'option_have_coretext',
        ]

    env = Environment(
        DEBUG_BUILD=GetOption('option_debug'),
        VERBOSE_COMPILE=GetOption('option_verbose'),
        RECONF=GetOption('option_reconfigure'),
        HARFBUZZ_CHECK=GetOption('option_check')
    )

    for option in harfbuzz_options:
        var_name = option.replace('option_', '').upper()
        env[var_name] = GetOption(option)

    if env['BUILD_UTILS']:
        env['HAVE_GLIB'] = True
        env['HAVE_FREETYPE'] = True

    if env['HAVE_GOBJECT']:
        env['HAVE_GLIB'] = True

    if env['HAVE_INTROSPECTION']:
        env['HAVE_GOBJECT'] = True
        env['HAVE_GLIB'] = True

    if env['HARFBUZZ_CHECK']:
        env['BUILD_SHARED'] = True
        env['BUILD_UTILS'] = True
        env['BUILD_UCDN'] = True
        env['HAVE_ICU'] = True
        env['HAVE_GLIB'] = True
        env['HAVE_GOBJECT'] = True
        env['HAVE_INTROSPECTION'] = True
        env['HAVE_FREETYPE'] = True
        env['HAVE_GRAPHITE2'] = True
        if sys.platform == 'win32':
            env['HAVE_UNISCRIBE'] = True
            env['HAVE_DIRECTWRITE'] = True
        if sys.platform == 'darwin':
            env['HAVE_CORETEXT'] = True

    env['PROJECT_DIR'] = os.path.abspath(Dir('.').abspath).replace('\\', '/')
    env['BUILD_DIR'] = 'build'

    SetBuildJobs(env)
    GetHarfbuzzVersion(env)

    project_sources = []
    project_headers = []
    gobject_sources = []
    gobject_gen_sources = []
    hb_gobject_structs_headers = []
    hb_gobject_headers = []
    project_extra_sources = []
    hb_gobject_gen_headers = []
    subset_project_sources = []

    project_sources += (
        GetSources('HB_BASE_sources', 'repo/src/Makefile.sources') +
        GetSources('HB_BASE_RAGEL_GENERATED_sources', 'repo/src/Makefile.sources') +
        GetSources('HB_FALLBACK_sources', 'repo/src/Makefile.sources') +
        GetSources('HB_BASE_headers', 'repo/src/Makefile.sources'))

    subset_project_sources += GetSources('HB_SUBSET_sources', 'repo/src/Makefile.sources')

    if env['HAVE_FREETYPE']:
        if not FindFreetype(env, conf_dir=env['BUILD_DIR']):
            p.ErrorPrint(
                "Requested build with Freetype, but couldn't find it.")
            return
        env.Append(CPPDEFINES=[('HAVE_FREETYPE', '1')])
        project_sources += ['repo/src/hb-ft.cc']
        project_headers += ['repo/src/hb-ft.h']

    if env['HAVE_GRAPHITE2'] and FindGraphite2(env, conf_dir=env['BUILD_DIR']):
        env.Append(CPPDEFINES=['HAVE_GRAPHITE2'])
        project_sources += ['repo/src/hb-graphite2.cc']
        project_headers += ['repo/src/hb-graphite2.h']

    if env['BUILD_UCDN']:
        env.Append(
            CPPPATH=['repo/src/hb-ucdn'],
            CPPDEFINES=['HAVE_UCDN'])
        project_sources += ['repo/src/hb-ucdn.cc']
        project_extra_sources += GetSources('LIBHB_UCDN_sources',
                                            'repo/src/hb-ucdn/Makefile.sources')
    
    if env['HAVE_GLIB'] and FindGlib(env, conf_dir=env['BUILD_DIR']):
        env.Append(CPPDEFINES=['HAVE_GLIB'])
        project_sources += ['repo/src/hb-glib.cc']
        project_headers += ['repo/src/hb-glib.h']

    if env['HAVE_ICU'] and FindIcu(env, conf_dir=env['BUILD_DIR']):
        env.Append(CPPDEFINES=['HAVE_ICU'])
        project_sources += ['repo/src/hb-icu.cc']
        project_headers += ['repo/src/hb-icu.h']

    if sys.platform == 'darwin':
        if env['HAVE_CORETEXT']:
            env.Append(CPPDEFINES=['HAVE_CORETEXT'])
            project_sources += ['repo/src/hb-coretext.cc']
            project_headers += ['repo/src/hb-coretext.h']

    if sys.platform == 'win32':
        if env['HAVE_UNISCRIBE']:
            env.Append(
                CPPDEFINES=['HAVE_UNISCRIBE'],
                LIBS=[
                    'usp10',
                    'gdi32',
                    'rpcrt4'
                ])
            project_sources += ['repo/src/hb-uniscribe.cc']
            project_headers += ['repo/src/hb-uniscribe.h']

        if env['HAVE_DIRECTWRITE']:
            env.Append(
                CPPDEFINES=['HAVE_DIRECTWRITE'],
                LIBS=[
                    'dwrite',
                    'rpcrt4'
                ])
            project_sources += ['repo/src/hb-directwrite.cc']
            project_headers += ['repo/src/hb-directwrite.h']

    if env['HAVE_GOBJECT']:

        glibmkenum_path = None
        perl_path = None
        glibmkenum_cmd = None

        bin_ext = ""
        if sys.platform == 'win32':
            bin_ext = ".exe"

        for path in os.environ["PATH"].split(os.pathsep):
            if os.access(os.path.join(path, 'glib-mkenums' + bin_ext), os.X_OK):
                glibmkenum_path = os.path.join(path, 'glib-mkenums' + bin_ext)
            if os.access(os.path.join(path, 'perl' + bin_ext), os.X_OK):
                perl_path = os.path.join(path, 'perl' + bin_ext)

        if glibmkenum_path:
            process = subprocess.Popen(
                [sys.executable, glibmkenum_path, '--version'], stdout=subprocess.PIPE, stderr=subprocess.PIPE)
            stdout, stderr = process.communicate()
            if process.returncode == 0:
                glibmkenum_cmd = [sys.executable, glibmkenum_path]
            elif perl_path:
                process = subprocess.Popen(
                    [perl_path, glibmkenum_path, '--version'], stdout=subprocess.PIPE, stderr=subprocess.PIPE)
                stdout, stderr = process.communicate()
                if process.returncode == 0:
                    glibmkenum_cmd = [perl_path, glibmkenum_path]
        else:
            p.ErrorPrint(
                "Requested build with gobjects, but couldn't find glib-mkenums")
            return

        if not glibmkenum_cmd:
            p.ErrorPrint(
                "Requested build with gobjects, but couldn't find glib-mkenums")
            return

        gobject_sources += ['repo/src/hb-gobject-structs.cc']
        gobject_gen_sources += ['repo/src/hb-gobject-enums.cc']
        hb_gobject_structs_headers += ['repo/src/hb-gobject-structs.h']
        hb_gobject_headers += hb_gobject_structs_headers + \
            ['repo/src/hb-gobject.h']
        hb_gobject_gen_headers += ['repo/src/hb-gobject-enums.h']

        process = subprocess.Popen(glibmkenum_cmd + ['--template', 'repo/src/hb-gobject-enums.cc.tmpl', '--identifier-prefix', 'hb_',
                                                     '--symbol-prefix', 'hb_gobject'] + hb_gobject_headers + project_headers, stdout=subprocess.PIPE, stderr=subprocess.PIPE)
        
        stdout, stderr = process.communicate()
        if process.returncode == 0:
            f1 = open('repo/src/hb-gobject-enums.h', 'w')
            f1.write(stdout.decode('utf8').replace('_t_get_type',
                                    '_get_type').replace('_T (', ' ('))
            f1.close()
        if stderr:
            p.ErrorPrint(
                "Failed to run glib-mkenums: " + stderr.decode(utf8))

        process = subprocess.Popen(glibmkenum_cmd + ['--template', 'repo/src/hb-gobject-enums.cc.tmpl', '--identifier-prefix', 'hb_',
                                                     '--symbol-prefix', 'hb_gobject'] + hb_gobject_headers + project_headers, stdout=subprocess.PIPE, stderr=subprocess.PIPE)
        stdout, stderr = process.communicate()
        if process.returncode == 0:
            f1 = open('repo/src/hb-gobject-enums.cc', 'w')
            f1.write(stdout.decode('utf8').replace('_t_get_type',
                                    '_get_type').replace('_T (', ' ('))
            f1.close()
        if stderr:
            p.ErrorPrint(
                "Failed to run glib-mkenums: " + stderr.decode(utf8))

    if env['BUILD_SHARED']:
        if sys.platform == 'win32':
            env.Append(CPPDEFINES=['HB_DLL_EXPORT'])

    ConfigureEnv(env)

    if (env['HAVE_BSYMBOLIC'] and
        (sys.platform != 'win32' or 
        (env.Detect('gcc') and sys.platform == 'win32'))):

        env.Append(
            LINKFLAGS=['-Bsymbolic-functions'],
            CCFLAGS=[
                '-fno-rtti', 
                '-fno-exceptions'
            ],
        )



    progress = ProgressCounter()

    project_sources = [
        source for source in project_sources if source.endswith(".cc")]
    project_extra_sources = [
        source for source in project_extra_sources if source.endswith(".c")]
    subset_project_sources = [
        source for source in subset_project_sources if source.endswith(".cc")]
    
    static_env, harf_static = SetupBuildEnv(
        env,
        progress,
        'static',
        'harfbuzz',
        project_sources + project_extra_sources,
        env['BUILD_DIR'] + '/build_static',
        'deploy_static')

    shared_env, harf_shared = SetupBuildEnv(
        env,
        progress,
        'shared',
        'harfbuzz',
        project_sources + project_extra_sources,
        env['BUILD_DIR'] + '/build_shared',
        'deploy')

    subset_shared_env, subset_shared = SetupBuildEnv(
        env,
        progress,
        'shared',
        'harfbuzz-subset',
        subset_project_sources,
        env['BUILD_DIR'] + '/subset_shared',
        'deploy')

    subset_shared_env.Append(
        LIBS=['harfbuzz'],
        LIBPATH=[env['PROJECT_DIR'] + '/' + env['BUILD_DIR'] + '/build_shared'])

    if sys.platform != 'win32':
        subset_shared_env.Append(
            CCFLAGS=['-fvisibility-inlines-hidden'])

    subset_static_env, subset_static = SetupBuildEnv(
        env,
        progress,
        'static',
        'harfbuzz-subset',
        subset_project_sources,
        env['BUILD_DIR'] + '/subset_static',
        'deploy_static')

    subset_static_env.Append(
        LIBS=['harfbuzz'],
        LIBPATH=[env['PROJECT_DIR'] + '/' + env['BUILD_DIR'] + '/build_static'])

    tests = [
        'main',
        'test',
        'test-would-substitute', 
        'test-size-params',
        'test-buffer-serialize',
        'hb-ot-tag', 
        'test-unicode-ranges'
    ]

    for test in tests:
        test_env, test_bin = SetupBuildEnv(
            env,
            progress,
            'exec',
            test,
            ['repo/src/' + test + ".cc"],
            env['BUILD_DIR'] + '/tests',
            'deploy')
        if test == 'hb-ot-tag':
            test_env.Append(CPPDEFINES=['MAIN'])
        test_env.Append(
            LIBS=['harfbuzz'],
            LIBPATH=[env['PROJECT_DIR'] + '/' + env['BUILD_DIR'] + '/build_shared'])



    Progress(progress, interval=1)


def ConfigureEnv(env):

    p = ColorPrinter()

    if not env.GetOption('clean'):

        if env['RECONF']:
            os.remove(env['BUILD_DIR'] + '/build.conf')

        vars = Variables(env['BUILD_DIR'] + '/build.conf')

        # vars.Add('ZPREFIX', '' )
        # vars.Add('SOLO', '')
        # vars.Add('COVER', '')
        vars.Update(env)

        # if('--zprefix' in sys.argv): env['ZPREFIX'] = GetOption('option_zprefix')
        # if('--solo' in sys.argv):    env['SOLO']    = GetOption('option_solo')
        # if('--cover' in sys.argv):   env['COVER']   = GetOption('option_cover')

        vars.Save(env['BUILD_DIR'] + '/build.conf', env)

        configureString = ""
        # if env['ZPREFIX']: configureString += "--zprefix "
        # if env['SOLO']:    configureString += "--solo "
        # if env['COVER']:   configureString += "--cover "

        if configureString == "":
            configureString = " Configuring... "
        else:
            configureString = " Configuring with " + configureString

        p.InfoPrint(configureString)

        # ruins logs so turning it off
        #SCons.Script.Main.progress_display.set_mode(1)
        
        conf = Configure(env, conf_dir=env['BUILD_DIR'] + "/conf_tests", log_file=env['BUILD_DIR'] + "/conf.log",
                        custom_tests={  
                            'CheckLargeFile64': CheckLargeFile64,
                            'CheckFseeko': CheckFseeko,
                            'CheckSizeT': CheckSizeT,
                            'CheckSizeTLongLong': CheckSizeTLongLong,
                            'CheckSizeTPointerSize': CheckSizeTPointerSize,
                            'CheckSharedLibrary': CheckSharedLibrary,
                            'CheckUnistdH': CheckUnistdH,
                            'CheckSolarisAtomics': CheckSolarisAtomics,
                            'CheckIntelAtomicPrimitives': CheckIntelAtomicPrimitives,
                            'CheckFunc' : CheckFunc,
                            'CheckHeader' : CheckHeader,
                            'CheckBSymbolic' : CheckBSymbolic
                        })
     
        # with open('repo/zconf.h.in', 'r') as content_file:
        #    conf.env["ZCONFH"] = str(content_file.read())

        conf.CheckSharedLibrary()

        if conf.CheckBSymbolic():
            env['HAVE_BSYMBOLIC'] = True
        else:
            env['HAVE_BSYMBOLIC'] = False

        checks_funcs = [
            'atexit',
            'mprotect',
            'sysconf',
            'getpagesize',
            'mmap',
            'isatty',
            'newlocale',
            'strtod_l',
            'round'
        ]

        if env.get('HB_HAVE_FREETYPE'):
            checks_funcs += [
                'FT_Get_Var_Blend_Coordinates',
                'FT_Set_Var_Blend_Coordinates',
                'FT_Done_MM_Var'
            ]

        for func in checks_funcs:
            have_func = 'HAVE_' + func.upper()
            if(conf.CheckFunc(func)):
                env[have_func] = True
                env.Append(CPPDEFINES=[have_func])
            else:
                env[have_func] = False

        check_headers = [
            'unistd.h',
            'sys/mman.h',
            'xlocale.h',
            'stdbool.h'
        ]

        for header in check_headers:
            have_header = 'HAVE_' + header.upper().replace('/', '_').replace('.', '_')
            if(conf.CheckHeader(header, language='C')):
                env[have_header] = True
                env.Append(CPPDEFINES=[have_header])
            else:
                env[have_header] = False

        # conf.CheckExternalNames()
        if not conf.CheckSizeT():
            conf.CheckSizeTPointerSize(conf.CheckSizeTLongLong())
        if not conf.CheckLargeFile64():
            conf.CheckFseeko()

        conf.CheckUnistdH()
        conf.CheckSolarisAtomics()
        conf.CheckIntelAtomicPrimitives()

        # ruins logs so turning it off
        #SCons.Script.Main.progress_display.set_mode(0)

        env = conf.Finish()

    if("linux" in platform.system().lower()):

        debugFlag = ""
        if(env['DEBUG_BUILD']):
            debugFlag = "-g"

        env.Append(CPPPATH=[
        ])

        env.Append(CCFLAGS=[
            debugFlag,
            '-O3',
            # '-fPIC',
            # "-rdynamic",
        ])

        env.Append(CXXFLAGS=[
            # "-std=c++11",
        ])

        env.Append(LDFLAGS=[
            debugFlag,
            # "-rdynamic",
        ])

        env.Append(LIBPATH=[
            # "/usr/lib/gnome-settings-daemon-3.0/",
        ])

        env.Append(LIBS=[
            'm',
        ])

    elif("darwin" in platform.system().lower()):
        print("XCode project not implemented yet")
    elif("win" in platform.system().lower()):

        degugDefine = 'NDEBUG'
        debugFlag = "/O2"
        degug = '/DEBUG:NONE'
        debugRuntime = "/MD"
        libType = "Release"
        if(env['DEBUG_BUILD']):
            degugDefine = 'DEBUG'
            debugFlag = "/Od"
            degug = '/DEBUG:FULL'
            debugRuntime = "/MDd"
            libType = 'Debug'

        env.Append(
            CCFLAGS=[
                '/wd4244',
                '/wd4267'
            ],
            CPPDEFINES=[
                '_CRT_SECURE_NO_WARNINGS',
                '_CRT_NONSTDC_NO_WARNINGS',
            ],

        )

    env.Append(
        CPPDEFINES=['HAVE_FALLBACK']
    )

    return env


def ConfigPlatformIDE(env, sourceFiles, headerFiles, resources, program):
    if platform == "linux" or platform == "linux2":
        print("Eclipse C++ project not implemented yet")
    elif("darwin" in platform.system().lower()):
        print("XCode project not implemented yet")
    elif("win" in platform.system().lower()):
        variantSourceFiles = []
        for file in sourceFiles:
            variantSourceFiles.append(re.sub("^build", "../Source", file))
        variantHeaderFiles = []
        for file in headerFiles:
            variantHeaderFiles.append(re.sub("^Source", "../Source", file))
        buildSettings = {
            'LocalDebuggerCommand': os.path.abspath(env['PROJECT_DIR']).replace('\\', '/') + "/build/MyLifeApp.exe",
            'LocalDebuggerWorkingDirectory': os.path.abspath(env['PROJECT_DIR']).replace('\\', '/') + "/build/",

        }
        buildVariant = 'Release|x64'
        cmdargs = ''
        if(env['DEBUG_BUILD']):
            buildVariant = 'Debug|x64'
            cmdargs = '--debug_build'
        env.MSVSProject(target='VisualStudio/MyLifeApp' + env['MSVSPROJECTSUFFIX'],
                        srcs=variantSourceFiles,
                        localincs=variantHeaderFiles,
                        resources=resources,
                        buildtarget=program,
                        DebugSettings=buildSettings,
                        variant=buildVariant,
                        cmdargs=cmdargs)
    return env


def GetSources(var, file):
    with open(file) as f:
        sources = re.search(
            r'^' + var + r'\s=\s([^$]+)\\$', f.read(), re.MULTILINE).group(1).split('\\')
        sources = [line.strip() for line in sources if line != ""]
        sources = [os.path.dirname(file) + '/' + line for line in sources]
        return sources


def GetHarfbuzzVersion(env=None):
    with open('repo/configure.ac') as f:
        match = re.search(
            r'\[(([0-9]+)\.([0-9]+)\.([0-9]+))\]', f.read(), re.MULTILINE)
        if env:
            env['HB_VERSION'] = match.group(1).strip()
            env['HB_VERSION_MAJOR'] = match.group(2).strip()
            env['HB_VERSION_MINOR'] = match.group(3).strip()
            env['HB_VERSION_MICRO'] = match.group(4).strip()
        return match.group(1).strip()


def SetupInstalls(env):

    return env


def SetupOptions():

    AddOption(
        '--have-freetype',
        dest='option_have_freetype',
        action='store_true',
        default=False,
        help='Enable freetype interop helpers'
    )

    AddOption(
        '--have-graphite2',
        dest='option_have_graphite2',
        action='store_true',
        default=False,
        help='Enable Graphite2 complementary shaper'
    )

    AddOption(
        '--no-builtin-ucdn',
        dest='option_build_ucdn',
        action='store_false',
        metavar='BOOL',
        default=True,
        help="Don't use HarfBuzz provided UCDN"
    )

    AddOption(
        '--build-utils',
        dest='option_build_utils',
        action='store_true',
        metavar='BOOL',
        default=False,
        help='Build harfbuzz utils, needs cairo, freetype, and glib properly be installed'
    )

    AddOption(
        '--have-glib',
        dest='option_have_glib',
        action='store_true',
        metavar='BOOL',
        default=False,
        help='Enable glib unicode functions'
    )

    AddOption(
        '--have-icu',
        dest='option_have_icu',
        action='store_true',
        metavar='BOOL',
        default=False,
        help='Enable icu unicode functions'
    )

    AddOption(
        '--no-build-subset',
        dest='option_build_subset',
        action='store_false',
        metavar='BOOL',
        default=True,
        help="Don't build harfbuzz-subset"
    )

    AddOption(
        '--no-build-tests',
        dest='option_build_tests',
        action='store_false',
        metavar='BOOL',
        default=True,
        help="Don't Build harfbuzz tests"
    )

    AddOption(
        '--no-shared-libs',
        dest='option_build_shared',
        action='store_false',
        metavar='BOOL',
        default=True,
        help="Don't build shared libraries"
    )

    AddOption(
        '--have-gobject',
        dest='option_have_gobject',
        action='store_true',
        metavar='BOOL',
        default=False,
        help='Enable GObject Bindings'
    )

    AddOption(
        '--have-introspection',
        dest='option_have_introspection',
        action='store_true',
        metavar='BOOL',
        default=False,
        help='Enable building introspection (.gir/.typelib) files'
    )

    AddOption(
        '--check',
        dest='option_check',
        action='store_true',
        metavar='BOOL',
        default=False,
        help='Do a configuration suitable for testing (shared library and enable all options)'
    )

    AddOption(
        '--verbose',
        dest='option_verbose',
        action='store_true',
        metavar='BOOL',
        default=False,
        help='View compiler output.'
    )

    AddOption(
        '--debug-build',
        dest='option_debug',
        action='store_true',
        metavar='BOOL',
        default=False,
        help='Build Debug'
    )

    AddOption(
        '--reconfigure',
        dest='option_reconfigure',
        action='store_true',
        metavar='BOOL',
        default=False,
        help='Clear out configuration and reconfigure with new options.'
    )

    if sys.platform == 'win32':
        AddOption(
            '--have-uniscribe',
            dest='option_have_uniscribe',
            action='store_true',
            metavar='BOOL',
            default=False,
            help='Enable Uniscribe shaper backend on Windows'
        )

        AddOption(
            '--have-directwrite',
            dest='option_have_directwrite',
            action='store_true',
            metavar='BOOL',
            default=False,
            help='Enable DirectWrite shaper backend on Windows'
        )

    if sys.platform == 'darwin':
        AddOption(
            '--have-coretext',
            dest='option_have_coretext',
            action='store_true',
            metavar='BOOL',
            default=False,
            help='Enable CoreText shaper backend on macOS'
        )

   
    # ruins logs so turning it off
    #if(not GetOption('option_verbose')):
        #scons_ver = SCons.__version__
        #if int(scons_ver[0]) >= 3:
        #    SetOption('silent', 1)
        #SCons.Script.Main.progress_display.set_mode(0)


CreateNewEnv()
