#!/bin/bash

#=============================================================================
#
# s-audit.sh
# ----------
#
# Audit a machine to find out information about the hardware and software
# it's running. Works on all Solaris versions from 2.6 up.
#
# Can be run as any user, but a full audit requires root privileges.
# Currently is not RBAC aware.
#
# For usage information run
#
#  s-audit.sh -h
#
# For documentation, license, changelog and more please see
#
#   http://snltd.co.uk/s-audit
#
# v3.3 (c) 2011-2012 SNLTD.f
#
#=============================================================================

#-----------------------------------------------------------------------------
# VARIABLES

PATH=/bin:/usr/bin

WSP="   "
	# A space and a literal tab. ksh88 won't accept \t


# Set a big PATH. This should try to include everywhere we expect to find
# software. SPATH is for quicker searching later.

SPATH="/bin /usr/bin /usr/sbin /usr/lib/ssh /usr/xpg6/bin /usr/sun/bin /etc \
	$(find /usr/*/*bin /usr/local/*/*bin /opt/*/*bin /usr/*/libexec \
	/usr/local/*/libexec /opt/*/libexec /usr/postgres/*/bin \
	/opt/local/*/bin /*/bin -prune \
	2>/dev/null) /usr/netscape/suitespot /opt/VirtualBox"

PATH=$(echo "$SPATH" | tr " " : | tr "\n" ":")
#**#PATH=$(echo "$SPATH" | tr " " : | tr "\n" ":")

egrep --version >/dev/null 2>&1 && EGS="egrep -s" || EGS="ggrep -q"

L_TOOL_TESTS="openssl rsync mysql_c postgres_c sqlplus svn_c java perl php_cmd
	python ruby node cc gcc pca nettracker saudit scat explorer jass jet"
G_TOOL_TESTS="sccli sneep vts $L_TOOL_TESTS"

#-- GENERAL FUNCTIONS --------------------------------------------------------


function find_bins
{
	# Searches the entire PATH for a list of unique binaries with the
	# name(s) given by the supplied args. Links are expanded and target
	# inodes compared, so only unique programs are found. When multiple
	# paths to the same file are found, the shortest is displayed
	prog=$1
	tgt=$(whereis -b $prog | awk '{print$2}')
	[[ "$tgt" != "" ]] && echo "$tgt" || \
	for d in $SPATH
	do
		PATH=$d
		which $prog 2>/dev/null
	done | sort | uniq
}


#-- TOOL TESTS --------------------------------------------------------------

function get_openssl
{
	# Gets the version of any OpenSSL binary it finds. This is probably a
	# good indicator of the OpenSSL libraries any SSL enabled software is
	# running, but not guaranteed.

	for BIN in $(find_bins openssl)
	do
		OSSL_VER=$($BIN version)
		OSSL_VER=${OSSL_VER#* }
		echo "OpenSSL@$BIN" ${OSSL_VER%% *}
	done
}

function get_rsync
{
	# rsync's version reporting is, err, thorough

	for BIN in $(find_bins rsync)
	do
		echo "rsync@$BIN" $($BIN --version | \
		sed '1!d;s/^.*sion \([^ ]*\) ..*$/\1/')
	done
}

function get_mysql_c
{
	# Get the version of any MySQL client software we find

	for BIN in $(find_bins mysql)
	do
		echo "MySQL client@$BIN" $($BIN -V | sed 's/^.*rib \([^ -,]*\).*$/\1/')
	done
}

function get_postgres_c
{
	# Get the version of the Postgres client

	for BIN in $(find_bins psql)
	do
		echo "Postgres client@$BIN" $($BIN --version | sed  's/.* //;q')
	done
}

function get_sqlplus
{
	# Get verion of SQL*Plus

	[[ -f ${ORACLE_HOME}/bin/sqlplus ]] \
		&& echo "sqlplus" $(ORACLE_HOME=$ORACLE_HOME \
		${ORACLE_HOME}/bin/sqlplus -\? \
		| sed -n '/SQL/s/^.*Release \([^ ]*\).*$/\1/p')
}

function get_svn_c
{
	# Get the version of Subversion client software

	for BIN in $(find_bins svn)
	do
		echo "svn client@$BIN" $($BIN --version --quiet)
	done
}

function get_java
{
	# Get the version of Java on this box. Nothing's ever straightforward
	# with Java is it? If there's a javac, assume it's a JDK.

	for BIN in $(find_bins java)
	do
		JV=$($BIN -version 2>&1 | sed -n '1s/^.*"\([^"]*\)"/\1/p')
		[[ $JV == *JDK_* ]] && JV=${JV#*JDK_}
		[[ -x "${BIN}c" ]] && JEX="(JDK)" || JEX="(JRE)"
		echo "Java@$BIN" $JV $JEX
	done
}

# Get the versions of a few scripting languages.

function get_python
{
	for BIN in $(find_bins python)
	do
		PV=$($BIN -V 2>&1 )
		echo "Python@$BIN" ${PV#* }
	done
}

function get_ruby
{
	for BIN in $(find_bins ruby)
	do
		RV=$($BIN -v)
		RV=${RV#* }
		echo "ruby@$BIN" ${RV%% *}
	done
}

function get_node
{
	for BIN in $(find_bins node)
	do
		NV=$($BIN -v)
		echo "node@$BIN" ${NV#v}
	done
}

function get_perl
{
	# Get the version of Perl

	for BIN in $(find_bins perl)
	do
		PV="$($BIN -V:version 2>/dev/null)" && PV=${PV%\'*}
		echo "perl@$BIN" ${PV##*\'}
	done
}

function get_php_cmd
{
	# Command line PHP binary

	for BIN in $(find_bins php)
	do
		disp "PHP cmdline@$BIN" $($BIN -v | sed '1!d;s/PHP \([^ ]*\).*$/\1/')
	done
}

function get_cc
{
	# Gets the version and patch number of whatever Sun are calling their C
	# compiler this week. Definitely works for 5, 7, 11 and 12.

	for BIN in $(find_bins cc)
	do
		# Is this really Sun CC?

		cc 2>&1 | $EGS "^usage" || continue

		CCDIR=${BIN%/*}
		CC_LIST="(C"
		CC_INFO=$($BIN -V 2>&1 | sed 1q)

		[[ "$CC_INFO" == *Forte* ]] && FN=6 || FN=4

		CC_MAJ_VER=$(print $CC_INFO | cut -d\  -f$FN)

		if [[ $CC_INFO == *Patch* ]]
		then
			CC_MIN_VER=${CC_INFO% *}
			CC_MIN_VER=" ${CC_MIN_VER##* }"
		elif print $CC_INFO | $EGS "/[0-9][0-9]$"
		then
			CC_MIN_VER=" ${CC_INFO##* }"
		fi

		# Look to see what options we have. C/C++/Fortran/IDE

		[[ -x ${CCDIR}/CC ]] && CC_LIST="${CC_LIST}, C++"
		[[ -x ${CCDIR}/f77 ]] && CC_LIST="${CC_LIST}, Fortran"
		[[ -x ${CCDIR}/sunstudio ]] && CC_LIST="${CC_LIST}, IDE"
		CC_LIST="${CC_LIST})"

		echo "Sun CC@$BIN" ${CC_MAJ_VER}$CC_MIN_VER $CC_LIST
	done
}

function get_gcc
{
	# Get the version of GCC on this box and find out what languages it
	# supports. Doesn't understand ADA, because I don't.

	for BIN in $(find_bins gcc)
	do
		GCCDIR=${BIN%/*}
		GCC_LIST="(C"
		GCC_VER=$($BIN -dumpversion)
		[[ -x ${GCCDIR}/g++ ]] && GCC_LIST="${GCC_LIST}, C++"
		[[ -x ${GCCDIR}/gfortran ]] && GCC_LIST="${GCC_LIST}, Fortran"
		[[ -x ${GCCDIR}/gcj ]] && GCC_LIST="${GCC_LIST}, Java"
		$BIN --help=objc >/dev/null 2>&1 && GCC_LIST="${GCC_LIST}, Obj-C"
		$BIN --help=objc++ >/dev/null 2>&1 && GCC_LIST="${GCC_LIST}, Obj-C++"
		GCC_LIST="${GCC_LIST})"

		echo "GCC@$BIN" $GCC_VER $GCC_LIST
	done
}
get_openssl 
get_rsync 
get_mysql_c 
get_postgres_c 
get_sqlplus 
get_svn_c 
get_java 
get_perl 
get_php_cmd
get_python
get_ruby 
get_node
get_cc
get_gcc
