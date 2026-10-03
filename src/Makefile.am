# SPDX-License-Identifier: GPL-3.0-or-later
# SPDX-FileCopyrightText: 2026 Claire Ivanenka <claire@gnu-ai.org>

bin_PROGRAMS = inference-translator

# --- Phase 0: the compilable skeleton --------------------------------
#
# The translator /inference (prompt -> structured request, the trio
# request/status/result of PLAN.md section 3.3) arrives with phase 1
# built on trivfs; the interactive client `inference` with phase 2.
# The skeleton answers the GNU base commands so that the binary is
# inspectable on any POSIX system, Hurd or not.

inference_translator_SOURCES = main.c

AM_CPPFLAGS = -I$(top_srcdir)
AM_CFLAGS = $(C23_CFLAGS) -Wall -Wextra
