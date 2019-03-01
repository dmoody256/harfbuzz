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
from BuildUtils.FindPackages import FindFreetype, FindGraphite2
from BuildUtils.ConfigureChecks import *


def CreateNewEnv():

    p = ColorPrinter()
    SetupOptions()

    env = Environment(
        DEBUG_BUILD=GetOption('option_debug'),
        VERBOSE_COMPILE=GetOption('option_verbose'),
    )

    env['PROJECT_DIR'] = os.path.abspath(Dir('.').abspath).replace('\\', '/')
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

    project_sources += GetSources('HB_BASE_sources',
                                  'repo/src/Makefile.sources')
    project_sources += GetSources('HB_BASE_RAGEL_GENERATED_sources',
                                  'repo/src/Makefile.sources')
    project_sources += GetSources('HB_FALLBACK_sources',
                                  'repo/src/Makefile.sources')
    project_headers += GetSources('HB_BASE_headers',
                                  'repo/src/Makefile.sources')

    if GetOption('option_have_freetype'):
        if not FindFreetype(env):
            p.ErrorPrint(
                "Requested build with Freetype, but couldn't find it.")
            return
        env.Append(CPPDEFINES=[('HAVE_FREETYPE', '1')])
        project_sources += ['repo/src/hb-ft.cc']
        project_headers += ['repo/src/hb-ft.h']

    if GetOption('option_have_graphite2') and FindGraphite2(env):
        env.Append(CPPDEFINES=['HAVE_GRAPHITE2'])
        project_sources += ['repo/src/hb-graphite2.cc']
        project_headers += ['repo/src/hb-graphite2.h']

    if GetOption('option_builtin_ucdn'):
        env.Append(
            CPPPATH=['repo/src/hb-ucdn'],
            CPPDEFINES=['HAVE_UCDN'])
        project_sources += ['repo/src/hb-ucdn.cc']
        project_extra_sources += GetSources('LIBHB_UCDN_sources',
                                            'repo/src/hb-ucdn/Makefile.sources')

    if GetOption('option_have_glib') and FindGlib(env):
        env.Append(CPPDEFINES=['HAVE_GLIB'])
        project_sources += ['repo/src/hb-glib.cc']
        project_headers += ['repo/src/hb-glib.h']

    if GetOption('option_have_icu') and FindIcu(env):
        env.Append(CPPDEFINES=['HAVE_ICU'])
        project_sources += ['repo/src/hb-icu.cc']
        project_headers += ['repo/src/hb-icu.h']

    if sys.platform == 'darwin':
        if GetOption('option_have_coretext'):
            env.Append(CPPDEFINES=['HAVE_CORETEXT'])
            project_sources += ['repo/src/hb-coretext.cc']
            project_headers += ['repo/src/hb-coretext.h']

    if sys.platform == 'win32':
        if GetOption('option_have_uniscribe'):
            env.Append(
                CPPDEFINES=['HAVE_UNISCRIBE'],
                LIBS=[
                    'usp10',
                    'gdi32',
                    'rpcrt4'
                ])
            project_sources += ['repo/src/hb-uniscribe.cc']
            project_headers += ['repo/src/hb-uniscribe.h']

        if GetOption('option_have_directwrite'):
            env.Append(
                CPPDEFINES=['HAVE_DIRECTWRITE'],
                LIBS=[
                    'dwrite',
                    'rpcrt4'
                ])
            project_sources += ['repo/src/hb-directwrite.cc']
            project_headers += ['repo/src/hb-directwrite.h']

    if GetOption('option_have_gobject'):
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
                    glibmkenum_cmd = [sys.perl_path, glibmkenum_path]

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

        process = subprocess.Popen(glibmkenum_cmd + ['--template=repo/src/hb-gobject-enums.cc.tmpl', '--identifier-prefix', 'hb_',
                                                     '--symbol-prefix', 'hb_gobject'] + hb_gobject_headers + project_headers, stdout=subprocess.PIPE, stderr=subprocess.PIPE)
        stdout, stderr = process.communicate()
        if process.returncode == 0:
            f1 = open('repo/src/hb-gobject-enums.h', 'w')
            f1.write(stdout.replace('_t_get_type',
                                    '_get_type').replace('_T (', ' ('))
            f1.close()

        process = subprocess.Popen(glibmkenum_cmd + ['--template=repo/src/hb-gobject-enums.cc.tmpl', '--identifier-prefix', 'hb_',
                                                     '--symbol-prefix', 'hb_gobject'] + hb_gobject_headers + project_headers, stdout=subprocess.PIPE, stderr=subprocess.PIPE)
        stdout, stderr = process.communicate()
        if process.returncode == 0:
            f1 = open('repo/src/hb-gobject-enums.cc', 'w')
            f1.write(stdout.replace('_t_get_type',
                                    '_get_type').replace('_T (', ' ('))
            f1.close()

    if GetOption('option_build_shared'):
        if sys.platform == 'win32':
            env.Append(CPPDEFINES=['HB_DLL_EXPORT'])

    ConfigureEnv(env)

    progress = ProgressCounter()

    project_sources = [
        source for source in project_sources if source.endswith(".cc")]
    project_extra_sources = [
        source for source in project_extra_sources if source.endswith(".c")]

    static_env, harf_static = SetupBuildEnv(
        env,
        progress,
        'static',
        'harfbuzz_static',
        project_sources + project_extra_sources,
        'build/build_static',
        'deploy')

    shared_env, harf_shared = SetupBuildEnv(
        env,
        progress,
        'shared',
        'harfbuzz',
        project_sources + project_extra_sources,
        'build/build_shared',
        'deploy')

    Progress(progress, interval=1)


def ConfigureEnv(env):

    p = ColorPrinter()

    if not env.GetOption('clean'):

        if(GetOption('option_reconfigure')):
            os.remove('build.conf')

        vars = Variables('build.conf')

        # vars.Add('ZPREFIX', '' )
        # vars.Add('SOLO', '')
        # vars.Add('COVER', '')
        vars.Update(env)

        # if('--zprefix' in sys.argv): env['ZPREFIX'] = GetOption('option_zprefix')
        # if('--solo' in sys.argv):    env['SOLO']    = GetOption('option_solo')
        # if('--cover' in sys.argv):   env['COVER']   = GetOption('option_cover')

        vars.Save('build.conf', env)

        configureString = ""
        # if env['ZPREFIX']: configureString += "--zprefix "
        # if env['SOLO']:    configureString += "--solo "
        # if env['COVER']:   configureString += "--cover "

        if configureString == "":
            configureString = "Configuring... "
        else:
            configureString = "Configuring with " + configureString

        p.InfoPrint(configureString)

        SCons.Script.Main.progress_display.set_mode(1)

        conf = Configure(env, conf_dir="build/conf_tests", log_file="conf.log",
                         custom_tests={
                             'CheckLargeFile64': CheckLargeFile64,
                             'CheckFseeko': CheckFseeko,
                             'CheckSizeT': CheckSizeT,
                             'CheckSizeTLongLong': CheckSizeTLongLong,
                             'CheckSizeTPointerSize': CheckSizeTPointerSize,
                             'CheckSharedLibrary': CheckSharedLibrary,
                             'CheckUnistdH': CheckUnistdH,
                             'CheckSolarisAtomics': CheckSolarisAtomics,
                             'CheckIntelAtomicPrimitives': CheckIntelAtomicPrimitives
                         })

        # with open('repo/zconf.h.in', 'r') as content_file:
        #    conf.env["ZCONFH"] = str(content_file.read())

        conf.CheckSharedLibrary()

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
            if(conf.CheckCHeader(header)):
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

        SCons.Script.Main.progress_display.set_mode(0)

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
        dest='option_builtin_ucdn',
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

    if GetOption('option_build_utils'):
        SetOption('option_have_glib', True)
        SetOption('option_have_freetype', True)

    if GetOption('option_have_gobject'):
        SetOption('option_have_glib', True)

    if GetOption('option_have_introspection'):
        SetOption('option_have_gobject', True)
        SetOption('option_have_glib', True)

    if GetOption('option_check'):
        SetOption('option_build_shared', True)
        SetOption('option_build_utils', True)
        SetOption('option_builtin_ucdn', True)
        SetOption('option_have_icu', True)
        SetOption('option_have_glib', True)
        SetOption('option_have_gobject', True)
        SetOption('option_have_introspection', True)
        SetOption('option_have_freetype', True)
        SetOption('option_have_graphite2', True)
        if sys.platform == 'win32':
            SetOption('option_have_uniscribe', True)
            SetOption('option_have_directwrite', True)
        if sys.platform == 'darwin':
            SetOption('option_have_coretext', True)

    if(not GetOption('option_verbose')):
        scons_ver = SCons.__version__
        if int(scons_ver[0]) >= 3:
            SetOption('silent', 1)
        SCons.Script.Main.progress_display.set_mode(0)


CreateNewEnv()
