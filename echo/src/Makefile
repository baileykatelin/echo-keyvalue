IDIR 	= ../include
ODIR	= obj
LDIR	= ../lib
SHAREDLIBVER	= 1.0.1
SHAREDSO        = libkp_kvstore.so
SHAREDSONAME    = $(SHAREDSO).1
SHAREDLIBNAME	= $(SHAREDSO).$(SHAREDLIBVER)
STATICLIBNAME	= libkp_kvstore.a
VECTOR_DIR	= vector-cdds
INCL = -I$(VECTOR_DIR)/ -Ihash_table/ -Ithreadpool/ -I../include -Ileveldb/include
MALLOC_DIR	= ./malloc
MALLOC_LIBS	= $(MALLOC_DIR)/lib

CC 	= gcc
LEVELDBLIBS = -lleveldb
MASSTREELIBS = -lmasstree
KPKVLIBS = -lkp_kvstore
TCMALLOC_LIBS	= -ltcmalloc_minimal
JEMALLOC_LIBS	= -ljemalloc
CFLAGS 	= $(INCL) -D_GNU_SOURCE -g -Wall -fPIC -O3 
#CFLAGS 	= $(INCL) -D_GNU_SOURCE -g -Wall -fPIC -O3 -Wextra -pedantic
  # -pg enables profiling with gprof(1)
  # -D_GNU_SOURCE is needed for CPU_SET macros in evaluation.c, and also
  #   needed for lock debugging
  # -fPIC: position-independent-code, needed for shared library...

# We've set up the Makefiles to use tcmalloc by default; anything built in
# this Makefile using the LIBS below will use tcmalloc, and the leveldb
# Makefile has been modified as well. Unfortunately we didn't add the
# Cassandra / Thrift source to our repository because we're stupid, so for
# Cassandra we have to use a different technique (see below). If you get 
# tcmalloc errors while _compiling_, then it probably means that you need to
# run "make tcmalloc" first (I couldn't quickly figure out how to get this
# dependency to be checked and built automatically... bah). If you get
# tcmalloc not-found errors at _runtime_, then it probably means that you
# just need to run "source source-me" in this directory (in the same shell
# that you're executing the program in).
#
# To double-check that tcmalloc is actually being used, run these commands
# and look for libtcmalloc_minimal.so.4 to appear in the list of dynamically
# loaded libraries. If libtcmalloc doesn't appear in the output of all of
# these commands, then something is wrong. If ldd shows the library in the
# list but says that it can't find it, then source source-me.
#   ldd leveldb/lib/libleveldb.so
#   ldd evaluation
LIBS	= -L$(LDIR) -L./masstree/ -L./leveldb/lib/ -L$(MALLOC_LIBS) -lpthread -lrt -lm $(TCMALLOC_LIBS)
  # -lrt is for scheduler/timing stuff in evaluation.c

# To use tcmalloc with Cassandra, we can use the less-recommended technique
# of pre-loading the tcmalloc library to override calls to malloc in
# executables that weren't built with tcmalloc linked in. This technique
# is less recommended because it's just not as elegant and because it will
# impact all programs executed in the particular shell. It's good enough for
# now though. To pre-load tcmalloc for use with Cassandra, just source the
# special file that contains the correct LD_PRELOAD command:
#   source source-me-Cassandra-tcmalloc
# You can echo $LD_PRELOAD afterwards to make sure that it takes effect.

default: test_local_master evaluation
lib: kp_kvstore-static kp_kvstore-shared
all: test_local_master evaluation test_leveldb test_masstree

#################################################
DEPS 	= ./kp_kvstore_internal.h $(IDIR)/kp_kv_local.h $(IDIR)/kp_kv_master.h ./kp_macros.h ./kp_recovery.h kp_common.h PIN_hooks.h

$(ODIR)/%.o: %.c $(DEPS)
	$(CC) -c -o $@ $< $(CFLAGS)
#	cp $@ $(LDIR)

$(ODIR)/hash_table.o: hash_table/hash.c hash_table/hash.h hash_table/bitrotate.h hash_table/xalloc-oversized.h ./kp_macros.h ./kp_recovery.h kp_common.h
	$(CC) -c -o $@ $< $(CFLAGS)

$(ODIR)/vector.o: $(VECTOR_DIR)/vector.c $(VECTOR_DIR)/vector.h kp_recovery.h
	$(CC) -c -o $@ $< $(CFLAGS)

$(ODIR)/vector64.o: $(VECTOR_DIR)/vector64.c $(VECTOR_DIR)/vector64.h kp_recovery.h
	$(CC) -c -o $@ $< $(CFLAGS)

$(ODIR)/queue.o: threadpool/queue.c threadpool/queue.h threadpool/threadpool_macros.h kp_recovery.h
	$(CC) -c -o $@ $< $(CFLAGS)

$(ODIR)/threadpool.o: threadpool/threadpool.c threadpool/threadpool.h threadpool/threadpool_macros.h kp_recovery.h kp_common.h
	$(CC) -c -o $@ $< $(CFLAGS)

_OBJ 	= PIN_Hooks.o kp_kvstore.o kp_kv_local.o kp_kv_master.o vector.o vector64.o hash_table.o queue.o threadpool.o kp_recovery.o kp_common.o 
OBJ 	= $(patsubst %,$(ODIR)/%,$(_OBJ))

# This test file is old and probably not worth using anymore:
#dummytest: tests/dummy_code.c $(OBJ)
#	$(CC) -o $@ $^ $(CFLAGS) $(LIBS)

test_local_master: tests/test_local_master.c $(OBJ)
	$(CC) -o $@ $^ $(CFLAGS) $(LIBS)

#################################################

tcmalloc:
	time ./make-tcmalloc.sh

jemalloc:
	time ./make-jemalloc.sh
#################################################

# This MASSTREE* stuff builds masstree_client.o:
MASSTREEDIR 	= masstree

# This target should create libmasstree.so.1[.0.1] in $LDIR.
masstree_lib:
	make -C $(MASSTREEDIR) shared_lib

masstree_clean:
	make -C $(MASSTREEDIR) clean

test_masstree: test_masstree.c masstree_lib
	gcc -o $@ test_masstree.c $(CFLAGS) $(LIBS) $(MASSTREELIBS)

#################################################

# This LEVELDB* stuff builds leveldb_client.o:
LEVELDB_DEPS 	= ./kp_kvstore_internal.h kp_macros.h leveldb_client.h
$(ODIR)/%.o: %.c $(LEVELDB_DEPS)
	$(CC) -c -o $@ $< $(CFLAGS)
_LEVELDB_OBJ	= leveldb_client.o
LEVELDB_OBJ		= $(patsubst %,$(ODIR)/%,$(_LEVELDB_OBJ))

# This is kind of a mess...
.PHONY: clean leveldb_lib leveldb_clean
LEVELDBDIR	= leveldb

# This target should create libleveldb.so.1[.0.1] in $LDIR.
leveldb_lib:
	make -C $(LEVELDBDIR) shared_lib

leveldb_clean:
	make -C $(LEVELDBDIR) clean

test_leveldb: test_leveldb.c leveldb_lib $(LEVELDB_OBJ)
	gcc -o $@ test_leveldb.c $(LEVELDB_OBJ) $(CFLAGS) $(LIBS) $(LEVELDBLIBS)

#################################################
_EVOBJ	= $(_OBJ) evaluation/evaluation.o 
EVOBJ 	= $(patsubst %,$(ODIR)/%,$(_EVOBJ))

evaluation: kp_kvstore-shared evaluation/evaluation.o leveldb_lib $(LEVELDB_OBJ) masstree_lib
	$(CC) -o evaluation/evaluation evaluation/evaluation.o $(LEVELDB_OBJ) $(CFLAGS) $(LIBS) $(LEVELDBLIBS) $(MASSTREELIBS) $(KPKVLIBS)
evaluation-shared: kp_kvstore-shared evaluation/evaluation.o leveldb_lib $(LEVELDB_OBJ)
	$(CC) -o evaluation/evaluation evaluation/evaluation.o $(LEVELDB_OBJ) $(CFLAGS) $(LIBS) $(LEVELDBLIBS) $(KPKVLIBS)
#	(this was what I used at first for shared evaluation: _EVOBJ = evaluation.o)
evaluation-nonshared: $(EVOBJ) leveldb_lib $(LEVELDB_OBJ)
	$(CC) -o $@ $(EVOBJ) $(LEVELDB_OBJ) $(CFLAGS) $(LIBS) $(LEVELDBLIBS)


#################################################
_OBJFLUSH	= flush.o kp_timing.o
OBJFLUSH	= $(patsubst %,$(ODIR)/%,$(_OBJFLUSH))

#-lrt is needed for clock_gettime():
test_flush: tests/test_flush.c $(OBJFLUSH)
	$(CC) -o $@ $^ -g -Wall -O2 -lrt
	$(CC) -o tests/test_flush2 tests/test_flush2.c -g -Wall -O2 -lrt
test_flush_assembly: tests/test_flush.c
	$(CC) -S -o flush.s flush.c -g -Wall -O2 -lrt
	$(CC) -S -o tests/test_flush.s tests/test_flush.c -g -Wall -O2 -lrt
test_cas: tests/test_cas.c $(OBJ)
	$(CC) -o $@ $^ $(INCL) -D_GNU_SOURCE -g -Wall -fPIC -O0 $(LIBS)

_OBJRECOVERY	= kp_recovery.o
OBJRECOVERY		= $(patsubst %,$(ODIR)/%,$(_OBJRECOVERY))

test_recovery: tests/test_recovery.c $(OBJRECOVERY) $(ODIR)/vector.o
	$(CC) -o $@ $^ -g -Wall -O2 -I$(VECTOR_DIR)/

#################################################
_LIBOBJ 	= $(_OBJ)
LIBOBJ 	= $(patsubst %,$(ODIR)/%,$(_LIBOBJ))

# http://www.adp-gmbh.ch/cpp/gcc/create_lib.html
kp_kvstore-static: $(LIBOBJ)
	ar rcs $(ODIR)/$(STATICLIBNAME) $(LIBOBJ)
	cp $(ODIR)/$(STATICLIBNAME) $(LDIR)

kp_kvstore-shared: $(LIBOBJ)
	$(CC) -shared -Wl,-soname,$(SHAREDSONAME) -o $(SHAREDLIBNAME) $(LIBOBJ) -lc
	rm -f $(LDIR)/*.so*
	mv $(SHAREDLIBNAME) $(LDIR)
	ln -s $(LDIR)/$(SHAREDLIBNAME) $(LDIR)/$(SHAREDSONAME)
	ln -s $(LDIR)/$(SHAREDLIBNAME) $(LDIR)/$(SHAREDSO)

#################################################
clean: leveldb_clean
	rm -f $(ODIR)/*.o $(ODIR)/*.a *~ core $(INCDIR)/*~ $(SHAREDLIBNAME) $(STATICLIBNAME) $(LDIR)/$(SHAREDLIBNAME) $(LDIR)/$(STATICLIBNAME) dummytest evaluation/evaluation telescope telescope2 test_local_master gmon.out test_leveldb test_flush test_flush2 test_recovery $(LDIR)/*.so* test_cas

#LIBS	= -L$(HOME)/lib/ -L/usr/local/lib/ -L./leveldb/lib/ -L$(MALLOC_LIBS) -lpthread -lrt -lm
#LIBS	= -L$(HOME)/lib/ -L/usr/local/lib/ -L./leveldb/lib/ -L$(MALLOC_LIBS) -lpthread -lrt -lm $(JEMALLOC_LIBS)
