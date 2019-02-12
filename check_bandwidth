#!/usr/bin/env perl
use warnings;
use Carp;
use Getopt::Long;
use File::Basename;

sub usage {
    print <<"END_USAGE";
Usage:
    $0 [ options ]

Options:
    --interface - if not specified check all interfaces in up state
    --interval  - how long in seconds do the measurment; default 15
END_USAGE
    exit 1;
}

my ( $interface, $interval );
GetOptions(
    'interface=s' => \$interface,
    'interval=i'  => \$interval,
) || usage;

$interval  //= 15;
$interface //= 'all';

my @interfaces;

if ( $interface eq 'all' ) {
    my @all_interfaces = </sys/class/net/*>;
    @interfaces = map { basename $_ }
      grep { -d "$_/bonding" && `grep -i up $_/operstate` } @all_interfaces;

    # if bonding not found select all interfaces in up state
    @interfaces
      = map { basename $_ }
      grep { !-d "$_/bridge" && `grep -i up $_/operstate` } @all_interfaces
      if !@interfaces;

    # or try to select venet ones
    @interfaces = grep /^venet/, map { basename $_ }  @all_interfaces
      if !@interfaces;
}
else {
    @interfaces = split( / /, $interface );
}

my %data;
for my $i (@interfaces) {
    for my $d (qw( rx tx )) {
        my $bytes = `cat /sys/class/net/$i/statistics/${d}_bytes`;
        chomp $bytes;
        $data{$i}{$d} = $bytes;
    }
}

sleep $interval;

my $output;
my $perfdata;

for my $i ( keys %data ) {
    for my $d (qw( rx tx )) {
        my $bytes = `cat /sys/class/net/$i/statistics/${d}_bytes`;
        chomp $bytes;
        $data{$i}{$d} = $bytes - $data{$i}{$d};
    }
    my $iface_speed_bits = `expr \$(cat /sys/class/net/$i/speed 2>/dev/null) '*' 1000000 2>/dev/null`;
    chomp $iface_speed_bits;
    $iface_speed_bits //= 100000000;
    my $in_bits   = sprintf( "%d",   $data{$i}{rx} * 8 / $interval );
    my $out_bits  = sprintf( "%d",   $data{$i}{tx} * 8 / $interval );
    my $in_mbits  = sprintf( "%.2f", $in_bits / 1000000 );
    my $out_mbits = sprintf( "%.2f", $out_bits / 1000000 );
    $output .= "${i}_in=$in_mbits Mbit/s, ${i}_out=$out_mbits Mbit/s; ";
    $perfdata
      .= "${i}_in=$in_bits;;;;$iface_speed_bits ${i}_out=$out_bits;;;;$iface_speed_bits ";
}

if ( !$output ) {
    print "empty output\n";
    exit 1;
}

print "$output | $perfdata\n";
