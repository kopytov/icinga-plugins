#!/usr/bin/env perl

use strict;
use warnings;

use Getopt::Long;

my $sensors        = '/usr/bin/sensors';
my $sensors_detect = '/usr/sbin/sensors-detect';

my %E_STATE = (
    'OK'       => 0,
    'WARNING'  => 1,
    'CRITICAL' => 2,
    'UNKNOWN'  => 3,
);
my ( $warning, $critical, $sudo, $help );
GetOptions(
    'w|warning=i'    => \$warning,
    'c|critical=i'   => \$critical,
    'sensors'        => \$sensors,
    'sensors-detect' => \$sensors_detect,
    's|sudo'         => \$sudo,
    'h|help'         => \$help,
);
return usage() if $help;

sub usage {
    print <<"END_USAGE";
Usage:
    $0 [ options ]

Options:
    -w|--warning     - WARNING_CPU_TEMP. Default value
    -c|--critical    - CRITICAL_CPU_TEMP
    -s|--sudo        - Use sudo for sensors and sensors-detect commands
    -h|--help        - this help information

    --sensors        - Set path to sensors util. Default /usr/sbin/sensors
    --sensors-detect - Set path to sensors-detect util. Default /usr/sbin/sensors-detect

    WARNING_CPU_TEMP  - default value from sensors output, if not def = 80;
    CRITICAL_CPU_TEMP - default value from sensors output, if not def = 100;

    Plugin depends from lm_sensors package.
END_USAGE
    exit $E_STATE{'UNKNOWN'};
}

# validate args and prepare data
if ( $> and !$sudo ) {
    print "you are not root\n";
    exit $E_STATE{'UNKNOWN'};
}
my $no_bin;
if ( !-X $sensors ) {
    $no_bin .= "sensors not found, please set path to sensors\n";
}
if ( !-X $sensors_detect ) {
    $no_bin .= "sensors-detect not found, please set path to sensors-detect\n";
}
if ($no_bin){
    print "$no_bin\n";
    usage();
}
my $exit_status       = 'OK';
my $status_string     = '';
my $perfdata_string   = '';

my $reg_t  = qr/\+([\d\.]+)/;
my $reg_dg = qr/\xc2\xb0C/;
my $reg    = qr/$reg_t$reg_dg\s+?(\(high\s=\s$reg_t$reg_dg,\s+crit\s=\s$reg_t$reg_dg\))/;

sub return_status {
    my $request_status = shift;
    if ( $request_status eq 'WARNING' ) {
        return if $exit_status eq 'CRITICAL';
    }
    if ( $request_status eq 'UNKNOWN' ) {
        return if $exit_status eq 'WARNING';
        return if $exit_status eq 'CRITICAL';
    }
    $exit_status = $request_status;
}
sub make_sensors_conf {
    system "echo -e | sudo $sensors_detect > /dev/null";
}

# sensors | grep "°C" | sed -r 's/:[ ]+[+]?([0-9.]+)°C.*$/=\1/;s/ /_/g;
my @sensors_data = `$sensors | grep "°C\\|coretemp-isa"`;
if (!grep { /^(Physical|Core)/ } grep {/$reg/} @sensors_data){
    if ( -f '/etc/sysconfig/lm_sensors' ){
        my $conf = `cat /etc/sysconfig/lm_sensors`;
        if ($conf =~ m/Run sensors-detect to generate this config file/) {
            make_sensors_conf;
            @sensors_data = `$sensors | grep "°C\\|coretemp-isa"`;
        }
        elsif (!$sudo) {
            system 'mv /etc/sysconfig/lm_sensors{,.bkp}';
            make_sensors_conf;
            @sensors_data = `$sensors | grep "°C\\|coretemp-isa"`;
            system 'mv /etc/sysconfig/lm_sensors{.bkp,}';
        }
    }
    else{
        make_sensors_conf;
        @sensors_data = `$sensors | grep "°C\\|coretemp-isa"`;
    }
}

my %cpus;
my $cpu = 0;
for my $line ( @sensors_data ){
    my ($name, $data);
    ($name, $data) = split /:\s+/, $line;
    $name =~ s/\s+//g;

    $cpu = sprintf("%d", $1)
      if $line =~ m/coretemp-isa-/
      && $line =~ m/(\d+)$/;

    next if $name !~ m/^\s?(Physical|Core)/;

    if ( $cpus{$cpu}{$name} ){
        foreach my $cp (sort keys %cpus){
            if ($cpus{$cp}{$name}){
                $cpu ++;
                last;
            }
        }
    }
    $cpus{$cpu}{$name} = 1;

    my ($temp, $crit, $warn);
    $data =~ $reg;
    ($temp, $warn, $crit) = ( int $1, int $3, int $4);
    $warning  = $warning  ? $warning  :
      defined $warn ? $warn : 80;
    $critical = $critical ? $critical :
      defined $crit ? $crit : 100;

    $name = "cpu$cpu.". lc $name;
    $perfdata_string .= "$name=$temp;$warning;$critical;; ";
    if ( $temp >= $critical ){
        $status_string .= "$name: temperature $temp is critical\n";
        return_status('CRITICAL');
    }
    elsif ( $temp >= $warning ){
        $status_string .= "$name: temperature $temp is warning\n";
        return_status('WARNING');
    }
}

print "$exit_status";
print ": \n$status_string" if $status_string;
print "|$perfdata_string\n";
exit $E_STATE{$exit_status};

1;
